# SecureStorage (flutter_secure_storage)

## 개요

`flutter_secure_storage` 패키지를 사용하여 민감한 데이터를 플랫폼 네이티브 보안 저장소에 암호화하여 저장하는 방법입니다. iOS는 **Keychain**, Android는 **EncryptedSharedPreferences** (API 23+)를 사용합니다. 인증 토큰, 비밀번호, API 키 등 **보안이 필요한 문자열 데이터** 저장에 사용합니다.

## 사용 시점

- Access Token / Refresh Token 저장
- 사용자 비밀번호, PIN 코드 저장
- API 키, 클라이언트 시크릿 등 앱 인증 정보
- 생체 인증(Face ID/지문)과 연동한 보안 데이터 저장

## 사용하지 말아야 할 때

- 앱 설정(테마, 언어), UI 상태(탭 인덱스) → [SharedPreferences](shared-preferences.md) 사용 (암복호화 오버헤드 불필요)
- 대용량 데이터, 구조화된 데이터 → SQLite (`drift`) 사용
- 바이너리 데이터 (이미지, 파일) → `path_provider` + 파일 시스템 사용
- 빈번하게 읽고 쓰는 데이터 → SharedPreferences 또는 in-memory 캐시 사용 (SecureStorage는 암복호화로 인해 상대적으로 느림)

## 필수 구현

### 기본 사용법

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const secureStorage = FlutterSecureStorage();

// 저장
await secureStorage.write(key: 'access_token', value: accessToken);
await secureStorage.write(key: 'refresh_token', value: refreshToken);

// 조회 — 키가 없으면 null 반환
final String? token = await secureStorage.read(key: 'access_token');

// 삭제
await secureStorage.delete(key: 'access_token');

// 전체 삭제
await secureStorage.deleteAll();

// 키 존재 여부 확인
final bool hasToken = await secureStorage.containsKey(key: 'access_token');
```

### 플랫폼별 옵션 설정

```dart
const secureStorage = FlutterSecureStorage(
  aOptions: AndroidOptions(
    encryptedSharedPreferences: true, // EncryptedSharedPreferences 사용 (API 23+)
  ),
  iOptions: IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
    // first_unlock_this_device: 기기 첫 잠금 해제 후 접근 가능, 백업 제외
  ),
);
```

### 키 상수 관리

```dart
abstract final class SecureStorageKeys {
  static const accessToken = 'access_token';
  static const refreshToken = 'refresh_token';
  static const userPin = 'user_pin';
  static const encryptionKey = 'encryption_key';
}
```

### Repository 패턴으로 래핑

```dart
abstract interface class AuthTokenRepository {
  Future<String?> getAccessToken();
  Future<String?> getRefreshToken();
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  });
  Future<void> clearTokens();
  Future<bool> hasTokens();
}

@LazySingleton(as: AuthTokenRepository)
class AuthTokenRepositoryImpl implements AuthTokenRepository {
  final FlutterSecureStorage _secureStorage;

  AuthTokenRepositoryImpl(this._secureStorage);

  @override
  Future<String?> getAccessToken() async {
    return await _secureStorage.read(key: SecureStorageKeys.accessToken);
  }

  @override
  Future<String?> getRefreshToken() async {
    return await _secureStorage.read(key: SecureStorageKeys.refreshToken);
  }

  @override
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _secureStorage.write(
        key: SecureStorageKeys.accessToken,
        value: accessToken,
      ),
      _secureStorage.write(
        key: SecureStorageKeys.refreshToken,
        value: refreshToken,
      ),
    ]);
  }

  @override
  Future<void> clearTokens() async {
    await Future.wait([
      _secureStorage.delete(key: SecureStorageKeys.accessToken),
      _secureStorage.delete(key: SecureStorageKeys.refreshToken),
    ]);
  }

  @override
  Future<bool> hasTokens() async {
    return await _secureStorage.containsKey(key: SecureStorageKeys.accessToken);
  }
}
```

### DI 등록

```dart
@module
abstract class StorageModule {
  @lazySingleton
  FlutterSecureStorage get secureStorage => const FlutterSecureStorage(
        aOptions: AndroidOptions(encryptedSharedPreferences: true),
        iOptions: IOSOptions(
          accessibility: KeychainAccessibility.first_unlock_this_device,
        ),
      );
}
```

### Dio Interceptor에서 토큰 주입

```dart
@injectable
class AuthInterceptor extends Interceptor {
  final AuthTokenRepository _tokenRepository;

  AuthInterceptor(this._tokenRepository);

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _tokenRepository.getAccessToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }
}
```

### 핵심 규칙

1. **문자열만 저장 가능** — 객체 저장 시 JSON 직렬화 필요하지만, 가급적 토큰/키 같은 단일 문자열만 저장
2. **Android는 `encryptedSharedPreferences: true` 필수** — 기본값은 AES 기반이지만, EncryptedSharedPreferences가 더 안전하고 안정적
3. **iOS는 `KeychainAccessibility` 설정 필수** — 기본값(`unlocked`)은 화면 잠김 시 접근 불가로 백그라운드 토큰 갱신 실패 가능
4. **Repository 패턴으로 래핑** — 비즈니스 로직에서 FlutterSecureStorage 직접 접근 금지
5. **앱 삭제/재설치 시 Keychain 잔류 데이터 처리** — iOS에서 앱 삭제 후 재설치 시 Keychain 데이터가 남아있을 수 있음, 첫 실행 시 정리 로직 필요

## 아키텍처 / 구조

```
lib/
├── core/
│   └── storage/
│       └── secure_storage_keys.dart     # 키 상수 정의
├── feature/auth/
│   └── data/
│       └── repository/
│           └── auth_token_repository.dart  # 인터페이스 + 구현
├── core/
│   └── network/
│       └── interceptor/
│           └── auth_interceptor.dart     # 토큰 주입 인터셉터
└── di/
    └── storage_module.dart              # DI 모듈
```

## 안티패턴

### 1. 토큰을 SharedPreferences에 저장

**잘못된 코드:**

```dart
// ❌ 인증 토큰을 평문 저장소에 저장
final prefs = SharedPreferencesAsync();
await prefs.setString('access_token', token);
```

**왜 잘못되었는가:** SharedPreferences는 평문 XML/plist 파일로 저장됩니다. 루팅/탈옥된 기기에서 즉시 읽을 수 있으며, 백업에도 포함되어 토큰이 유출될 수 있습니다.

**올바른 방법:**

```dart
// ✅ SecureStorage로 암호화 저장
const secureStorage = FlutterSecureStorage();
await secureStorage.write(key: 'access_token', value: token);
```

### 2. 플랫폼 옵션 없이 기본값 사용

**잘못된 코드:**

```dart
// ❌ 플랫폼 옵션 없이 기본 인스턴스 사용
const storage = FlutterSecureStorage(); // 기본 옵션만 사용

await storage.write(key: 'token', value: token);
```

**왜 잘못되었는가:** Android에서 기본 AES 암호화는 일부 기기에서 키스토어 초기화 문제가 발생할 수 있습니다. iOS에서 기본 `accessibility`는 화면 잠김 시 접근 불가합니다.

**올바른 방법:**

```dart
// ✅ 플랫폼별 옵션 명시
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
  ),
);
```

### 3. 토큰 갱신 로직에서 race condition

**잘못된 코드:**

```dart
// ❌ 동시에 여러 요청이 토큰 갱신을 시도
@override
Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
  if (err.response?.statusCode == 401) {
    final refreshToken = await _tokenRepository.getRefreshToken();
    // 여러 요청이 동시에 여기 도달하면 중복 갱신 발생
    final newTokens = await _authApi.refresh(refreshToken!);
    await _tokenRepository.saveTokens(
      accessToken: newTokens.accessToken,
      refreshToken: newTokens.refreshToken,
    );
    // 원래 요청 재시도
  }
}
```

**왜 잘못되었는가:** 401 응답이 동시에 여러 개 발생하면, 각각 토큰 갱신을 시도하여 refresh token이 무효화될 수 있습니다.

**올바른 방법:**

```dart
// ✅ Completer로 토큰 갱신 직렬화
class AuthInterceptor extends QueuedInterceptor {
  Completer<String>? _refreshCompleter;

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      try {
        final newToken = await _refreshToken();
        // 원래 요청에 새 토큰 적용 후 재시도
        final options = err.requestOptions;
        options.headers['Authorization'] = 'Bearer $newToken';
        final response = await _dio.fetch(options);
        handler.resolve(response);
      } catch (e) {
        handler.reject(err);
      }
    } else {
      handler.next(err);
    }
  }

  Future<String> _refreshToken() async {
    // 이미 갱신 중이면 기존 Completer를 기다림
    if (_refreshCompleter != null) {
      return _refreshCompleter!.future;
    }

    _refreshCompleter = Completer<String>();
    try {
      final refreshToken = await _tokenRepository.getRefreshToken();
      final newTokens = await _authApi.refresh(refreshToken!);
      await _tokenRepository.saveTokens(
        accessToken: newTokens.accessToken,
        refreshToken: newTokens.refreshToken,
      );
      _refreshCompleter!.complete(newTokens.accessToken);
      return newTokens.accessToken;
    } catch (e) {
      _refreshCompleter!.completeError(e);
      rethrow;
    } finally {
      _refreshCompleter = null;
    }
  }
}
```

### 4. iOS 앱 재설치 시 Keychain 잔류 데이터 무시

**잘못된 코드:**

```dart
// ❌ 앱 시작 시 Keychain 정리 없이 바로 토큰 사용
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final token = await secureStorage.read(key: 'access_token');
  // iOS: 앱 삭제 후 재설치해도 이전 토큰이 남아있을 수 있음
  if (token != null) {
    // 유효하지 않은 이전 토큰으로 API 호출 시도 → 실패
  }
}
```

**왜 잘못되었는가:** iOS에서 앱을 삭제해도 Keychain 데이터는 삭제되지 않습니다. 재설치 시 이전 앱의 만료된 토큰이 남아있어 인증 실패가 발생합니다.

**올바른 방법:**

```dart
// ✅ 첫 실행 감지 후 Keychain 정리
Future<void> clearKeychainOnFirstRun() async {
  final prefs = SharedPreferencesAsync();
  final isFirstRun = await prefs.getBool('is_first_run') ?? true;

  if (isFirstRun) {
    await secureStorage.deleteAll(); // Keychain 잔류 데이터 정리
    await prefs.setBool('is_first_run', false);
  }
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await clearKeychainOnFirstRun(); // 앱 시작 시 실행
  // ...
}
```

## 의사결정 기준

### SecureStorage vs SharedPreferences

| 기준 | SecureStorage | SharedPreferences |
|---|---|---|
| 보안 | 플랫폼 암호화 (Keychain/Keystore) | 평문 저장 |
| 속도 | 느림 (암복호화 오버헤드) | 빠름 |
| 데이터 유형 | 문자열만 (String) | bool, int, double, String, List\<String\> |
| 용도 | 토큰, 비밀번호, API 키 | 앱 설정, UI 상태, 플래그 |
| 앱 삭제 시 (iOS) | Keychain에 잔류 가능 | 함께 삭제됨 |

**의사결정 규칙:** 데이터가 유출되었을 때 보안 위협이 되는가? → Yes면 SecureStorage, No면 SharedPreferences.

## 테스트 전략

Repository를 Fake로 대체하여 테스트합니다. FlutterSecureStorage는 플랫폼 의존성이므로 직접 테스트하지 않습니다.

```dart
class FakeAuthTokenRepository implements AuthTokenRepository {
  String? _accessToken;
  String? _refreshToken;

  @override
  Future<String?> getAccessToken() async => _accessToken;

  @override
  Future<String?> getRefreshToken() async => _refreshToken;

  @override
  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    _accessToken = accessToken;
    _refreshToken = refreshToken;
  }

  @override
  Future<void> clearTokens() async {
    _accessToken = null;
    _refreshToken = null;
  }

  @override
  Future<bool> hasTokens() async => _accessToken != null;
}

void main() {
  late AuthBloc bloc;
  late FakeAuthTokenRepository fakeTokenRepo;

  setUp(() {
    fakeTokenRepo = FakeAuthTokenRepository();
    bloc = AuthBloc(fakeTokenRepo);
  });

  tearDown(() => bloc.close());

  blocTest<AuthBloc, AuthState>(
    '로그아웃 시 토큰이 삭제된다',
    build: () {
      fakeTokenRepo.saveTokens(
        accessToken: 'test_access',
        refreshToken: 'test_refresh',
      );
      return bloc;
    },
    act: (bloc) => bloc.add(const LogoutRequested()),
    expect: () => [
      const AuthState(status: AuthStatus.unauthenticated),
    ],
    verify: (_) async {
      expect(await fakeTokenRepo.hasTokens(), false);
    },
  );
}
```

### Mock 규칙

- **Fake O**: `AuthTokenRepository` — Fake 구현으로 저장소 격리
- **Mock X**: `FlutterSecureStorage` — 플랫폼 의존성이므로 Repository 레이어에서 격리

## AI가 자주 하는 실수

### 1. 비민감 데이터를 SecureStorage에 저장

**AI가 생성하는 코드:**

```dart
// ❌ 모든 데이터를 SecureStorage에 저장
await secureStorage.write(key: 'theme_mode', value: 'dark');
await secureStorage.write(key: 'language', value: 'ko');
await secureStorage.write(key: 'onboarding_done', value: 'true');
await secureStorage.write(key: 'last_tab', value: '2');
```

**무엇이 잘못되었는가:** SecureStorage는 암복호화 오버헤드가 있어 느립니다. 비민감 데이터까지 SecureStorage에 저장하면 불필요한 성능 저하가 발생합니다. 또한 SecureStorage는 문자열만 지원하여 `bool`, `int` 등을 매번 파싱해야 합니다.

**올바른 접근법:**

```dart
// ✅ 민감도에 따라 저장소 분리
// 민감한 데이터 → SecureStorage
await secureStorage.write(key: 'access_token', value: token);

// 비민감 데이터 → SharedPreferences
final prefs = SharedPreferencesAsync();
await prefs.setString('theme_mode', 'dark');
await prefs.setBool('onboarding_done', true);
await prefs.setInt('last_tab', 2);
```

### 2. 플랫폼 옵션 누락

**AI가 생성하는 코드:**

```dart
// ❌ 옵션 없이 인스턴스 생성
final storage = const FlutterSecureStorage();

// 또는 Android 옵션만 설정하고 iOS 무시
final storage = const FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  // iOS 옵션 누락
);
```

**무엇이 잘못되었는가:** Android에서 `encryptedSharedPreferences: true` 누락 시 일부 기기에서 KeyStore 관련 크래시가 발생할 수 있습니다. iOS에서 `accessibility` 누락 시 기본값(`unlocked`)이 적용되어 화면이 잠긴 상태에서 백그라운드 토큰 갱신이 실패합니다.

**올바른 접근법:**

```dart
// ✅ 양 플랫폼 옵션 모두 명시
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(
    accessibility: KeychainAccessibility.first_unlock_this_device,
  ),
);
```

### 3. iOS Keychain 잔류 데이터 미처리

**AI가 생성하는 코드:**

```dart
// ❌ 앱 시작 시 바로 토큰 읽기
Future<void> initAuth() async {
  final token = await secureStorage.read(key: 'access_token');
  if (token != null) {
    // 토큰이 있으면 로그인 상태로 판단
    emit(state.copyWith(status: AuthStatus.authenticated));
  }
}
```

**무엇이 잘못되었는가:** iOS에서 앱 삭제 후 재설치 시 Keychain 데이터가 남아있어 만료된 토큰으로 인증 상태를 잘못 판단합니다.

**올바른 접근법:**

```dart
// ✅ 첫 실행 시 Keychain 정리 + 토큰 유효성 검증
Future<void> initAuth() async {
  await clearKeychainOnFirstRun();

  final token = await _tokenRepository.getAccessToken();
  if (token != null) {
    // 토큰 유효성 API로 검증 후 상태 결정
    final isValid = await _authApi.validateToken(token);
    if (isValid) {
      emit(state.copyWith(status: AuthStatus.authenticated));
    } else {
      await _tokenRepository.clearTokens();
      emit(state.copyWith(status: AuthStatus.unauthenticated));
    }
  }
}
```

## 참고 자료

- [flutter_secure_storage 패키지](https://pub.dev/packages/flutter_secure_storage) — 공식 패키지
- [iOS Keychain Services](https://developer.apple.com/documentation/security/keychain_services) — Apple 공식 문서
- [Android EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences) — Android 공식 문서
