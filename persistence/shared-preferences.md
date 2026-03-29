# SharedPreferences

## 개요

`shared_preferences` 패키지를 사용하여 앱의 간단한 키-값 데이터를 디바이스 로컬에 영구 저장하는 방법입니다. 내부적으로 iOS는 `NSUserDefaults`, Android는 `SharedPreferences`를 사용합니다. 앱 설정, 온보딩 완료 여부, 마지막 선택 값 등 **소량의 비민감 데이터**를 저장하는 데 적합합니다.

## 사용 시점

- 앱 설정값 저장 (테마 모드, 언어, 알림 on/off)
- 온보딩/튜토리얼 완료 플래그
- 마지막으로 선택한 탭 인덱스, 필터 조건 등 UI 상태 복원
- 간단한 캐시 플래그 (마지막 동기화 시간 등)

## 사용하지 말아야 할 때

- 토큰, 비밀번호, API 키 등 민감한 데이터 → [flutter_secure_storage](secure-storage.md) 사용
- 대용량 구조화 데이터 (수백 건 이상의 레코드) → SQLite (`drift`, `sqflite`) 사용
- 복잡한 관계형 데이터 또는 쿼리가 필요한 경우 → SQLite (`drift`) 사용
- 바이너리 데이터 (이미지, 파일) → `path_provider` + 파일 시스템 사용

## 필수 구현

### SharedPreferencesAsync 사용 (권장)

`shared_preferences` 3.x부터 `SharedPreferencesAsync`가 권장 API입니다. 기존 `SharedPreferences.getInstance()`는 레거시이며, 새 프로젝트에서는 `SharedPreferencesAsync`를 사용하세요.

```dart
import 'package:shared_preferences/shared_preferences.dart';

// SharedPreferencesAsync 인스턴스 획득
final prefs = SharedPreferencesAsync();

// 저장
await prefs.setBool('onboarding_completed', true);
await prefs.setString('selected_language', 'ko');
await prefs.setInt('last_tab_index', 2);
await prefs.setDouble('font_size', 16.0);
await prefs.setStringList('recent_searches', ['flutter', 'dart']);

// 조회 — 키가 없으면 null 반환
final bool? onboarded = await prefs.getBool('onboarding_completed');
final String? lang = await prefs.getString('selected_language');

// 삭제
await prefs.remove('last_tab_index');

// 전체 삭제
await prefs.clear();
```

### 키 상수 관리

매직 스트링을 방지하기 위해 키를 상수로 관리합니다.

```dart
abstract final class PrefsKeys {
  static const onboardingCompleted = 'onboarding_completed';
  static const selectedLanguage = 'selected_language';
  static const themeMode = 'theme_mode';
  static const lastSyncTime = 'last_sync_time';
}
```

### Repository 패턴으로 래핑

SharedPreferences를 직접 사용하지 말고, Repository로 래핑하여 추상화합니다.

```dart
abstract interface class AppSettingsRepository {
  Future<ThemeMode> getThemeMode();
  Future<void> setThemeMode(ThemeMode mode);
  Future<bool> isOnboardingCompleted();
  Future<void> setOnboardingCompleted(bool completed);
}

@LazySingleton(as: AppSettingsRepository)
class AppSettingsRepositoryImpl implements AppSettingsRepository {
  final SharedPreferencesAsync _prefs;

  AppSettingsRepositoryImpl(this._prefs);

  @override
  Future<ThemeMode> getThemeMode() async {
    final value = await _prefs.getString(PrefsKeys.themeMode);
    return switch (value) {
      'dark' => ThemeMode.dark,
      'light' => ThemeMode.light,
      _ => ThemeMode.system,
    };
  }

  @override
  Future<void> setThemeMode(ThemeMode mode) async {
    await _prefs.setString(PrefsKeys.themeMode, mode.name);
  }

  @override
  Future<bool> isOnboardingCompleted() async {
    return await _prefs.getBool(PrefsKeys.onboardingCompleted) ?? false;
  }

  @override
  Future<void> setOnboardingCompleted(bool completed) async {
    await _prefs.setBool(PrefsKeys.onboardingCompleted, completed);
  }
}
```

### DI 등록

```dart
@module
abstract class StorageModule {
  @lazySingleton
  SharedPreferencesAsync get sharedPreferencesAsync =>
      SharedPreferencesAsync();
}
```

### 핵심 규칙

1. **민감한 데이터를 저장하지 않음** — 토큰, 비밀번호, 개인정보는 반드시 `flutter_secure_storage` 사용
2. **키는 상수로 관리** — 매직 스트링 금지, `PrefsKeys` 클래스로 중앙 관리
3. **Repository 패턴으로 래핑** — 비즈니스 로직에서 SharedPreferences 직접 접근 금지
4. **SharedPreferencesAsync 사용** — 레거시 `SharedPreferences.getInstance()` 대신 새 API 사용
5. **enum 저장 시 `.name`으로 직렬화** — `toString()` 사용 금지 (`ThemeMode.dark` → `'dark'`)

## 아키텍처 / 구조

```
lib/
├── core/
│   └── storage/
│       └── prefs_keys.dart              # 키 상수 정의
├── feature/{domain}/
│   └── data/
│       └── repository/
│           └── app_settings_repository.dart  # 인터페이스 + 구현
└── di/
    └── storage_module.dart              # DI 모듈
```

## 안티패턴

### 1. 민감한 데이터를 SharedPreferences에 저장

**잘못된 코드:**

```dart
// ❌ 토큰을 SharedPreferences에 저장
final prefs = SharedPreferencesAsync();
await prefs.setString('access_token', token);
await prefs.setString('refresh_token', refreshToken);
```

**왜 잘못되었는가:** SharedPreferences는 평문으로 저장되며 루팅된 기기에서 쉽게 읽을 수 있습니다. 토큰 탈취로 이어질 수 있습니다.

**올바른 방법:**

```dart
// ✅ 민감한 데이터는 SecureStorage 사용
final secureStorage = FlutterSecureStorage();
await secureStorage.write(key: 'access_token', value: token);
await secureStorage.write(key: 'refresh_token', value: refreshToken);
```

### 2. JSON을 통째로 SharedPreferences에 저장

**잘못된 코드:**

```dart
// ❌ 복잡한 객체를 JSON 문자열로 직렬화하여 저장
final prefs = SharedPreferencesAsync();
final userJson = jsonEncode(user.toJson());
await prefs.setString('cached_user', userJson);

final cartJson = jsonEncode(cartItems.map((e) => e.toJson()).toList());
await prefs.setString('cart_items', cartJson);
```

**왜 잘못되었는가:** SharedPreferences는 단순 키-값 저장소입니다. 복잡한 구조화 데이터를 JSON으로 직렬화하면 파싱 오류, 마이그레이션 불가, 성능 저하가 발생합니다.

**올바른 방법:**

```dart
// ✅ 구조화 데이터는 SQLite(drift) 사용
@DriftDatabase(tables: [CartItems])
class AppDatabase extends _$AppDatabase {
  // ...
}

// SharedPreferences는 단순 플래그/설정값만 저장
await prefs.setBool('has_items_in_cart', true);
```

### 3. 비즈니스 로직에서 SharedPreferences 직접 사용

**잘못된 코드:**

```dart
// ❌ BLoC에서 SharedPreferences 직접 접근
class SettingsBloc extends Bloc<SettingsEvent, SettingsState> {
  Future<void> _onThemeChanged(ThemeChanged event, Emitter<SettingsState> emit) async {
    final prefs = SharedPreferencesAsync();
    await prefs.setString('theme_mode', event.themeMode.name);
    emit(state.copyWith(themeMode: event.themeMode));
  }
}
```

**왜 잘못되었는가:** 저장소 구현에 직접 의존하면 테스트 시 mock이 어렵고, 저장소 변경 시 BLoC 수정이 필요합니다.

**올바른 방법:**

```dart
// ✅ Repository 인터페이스를 통해 접근
@injectable
class SettingsBloc extends Bloc<SettingsEvent, SettingsState> {
  final AppSettingsRepository _settingsRepository;

  SettingsBloc(this._settingsRepository) : super(const SettingsState()) {
    on<ThemeChanged>(_onThemeChanged);
  }

  Future<void> _onThemeChanged(ThemeChanged event, Emitter<SettingsState> emit) async {
    await _settingsRepository.setThemeMode(event.themeMode);
    emit(state.copyWith(themeMode: event.themeMode));
  }
}
```

## 의사결정 기준

### SharedPreferences vs SecureStorage vs SQLite

| 기준        | SharedPreferences              | SecureStorage                   | SQLite (drift)              |
| ----------- | ------------------------------ | ------------------------------- | --------------------------- |
| 데이터 유형 | 단순 키-값 (bool, int, String) | 민감한 문자열 (토큰, 비밀번호)  | 구조화된 대용량 데이터      |
| 보안        | 평문 저장                      | 암호화 저장 (Keychain/Keystore) | 평문 (암호화 선택적)        |
| 성능        | 빠름 (소량 데이터)             | 느림 (암복호화 오버헤드)        | 빠름 (인덱스, 쿼리 최적화)  |
| 쿼리        | 키 기반 조회만                 | 키 기반 조회만                  | SQL 쿼리 지원               |
| 적합한 용도 | 앱 설정, UI 상태               | 인증 토큰, API 키               | 목록, 캐시, 오프라인 데이터 |

**의사결정 규칙:** 민감한 데이터면 SecureStorage, 구조화/대량 데이터면 SQLite, 단순 설정/플래그면 SharedPreferences.

## 테스트 전략

Repository를 Fake로 대체하여 테스트합니다. SharedPreferences 자체를 테스트할 필요는 없습니다.

```dart
class FakeAppSettingsRepository implements AppSettingsRepository {
  ThemeMode _themeMode = ThemeMode.system;
  bool _onboardingCompleted = false;

  @override
  Future<ThemeMode> getThemeMode() async => _themeMode;

  @override
  Future<void> setThemeMode(ThemeMode mode) async => _themeMode = mode;

  @override
  Future<bool> isOnboardingCompleted() async => _onboardingCompleted;

  @override
  Future<void> setOnboardingCompleted(bool completed) async =>
      _onboardingCompleted = completed;
}

void main() {
  late SettingsBloc bloc;
  late FakeAppSettingsRepository fakeRepository;

  setUp(() {
    fakeRepository = FakeAppSettingsRepository();
    bloc = SettingsBloc(fakeRepository);
  });

  tearDown(() => bloc.close());

  blocTest<SettingsBloc, SettingsState>(
    '테마 변경 시 상태가 업데이트된다',
    build: () => bloc,
    act: (bloc) => bloc.add(const ThemeChanged(themeMode: ThemeMode.dark)),
    expect: () => [
      const SettingsState(themeMode: ThemeMode.dark),
    ],
    verify: (_) async {
      // Repository에도 저장되었는지 확인
      expect(await fakeRepository.getThemeMode(), ThemeMode.dark);
    },
  );
}
```

### Mock 규칙

- **Fake O**: `AppSettingsRepository` — Fake 구현으로 저장소 격리
- **Mock X**: `SharedPreferencesAsync` — 플랫폼 의존성이므로 Repository 레이어에서 격리

## AI가 자주 하는 실수

### 1. 레거시 API 사용 (SharedPreferences.getInstance)

**AI가 생성하는 코드:**

```dart
// ❌ 레거시 동기 캐싱 API 사용
final prefs = await SharedPreferences.getInstance();
prefs.setString('key', 'value'); // Future를 await하지 않음
final value = prefs.getString('key'); // 동기처럼 사용
```

**무엇이 잘못되었는가:** `SharedPreferences.getInstance()`는 레거시 API입니다. 3.x에서도 동작하지만, 내부적으로 모든 데이터를 메모리에 캐싱하여 메모리 낭비가 발생하고, `setX()`의 반환값(`Future<bool>`)을 무시하면 저장 실패를 놓칠 수 있습니다.

**올바른 접근법:**

```dart
// ✅ SharedPreferencesAsync 사용
final prefs = SharedPreferencesAsync();
await prefs.setString('key', 'value'); // 항상 await
final value = await prefs.getString('key'); // 비동기 조회
```

### 2. enum을 toString()으로 저장

**AI가 생성하는 코드:**

```dart
// ❌ toString()으로 직렬화
await prefs.setString('theme', ThemeMode.dark.toString());
// 저장되는 값: "ThemeMode.dark"

// 역직렬화 시 파싱이 복잡해짐
final themeStr = await prefs.getString('theme');
final theme = ThemeMode.values.firstWhere(
  (e) => e.toString() == themeStr, // "ThemeMode.dark"과 비교
);
```

**무엇이 잘못되었는가:** `toString()`은 `"ThemeMode.dark"` 형식으로 저장되어 불필요하게 길고, enum 클래스명이 변경되면 역직렬화가 깨집니다.

**올바른 접근법:**

```dart
// ✅ .name으로 직렬화
await prefs.setString('theme', ThemeMode.dark.name);
// 저장되는 값: "dark"

// 역직렬화 — Dart 2.15+ pattern matching
final themeStr = await prefs.getString('theme');
final theme = switch (themeStr) {
  'dark' => ThemeMode.dark,
  'light' => ThemeMode.light,
  _ => ThemeMode.system,
};
```

### 3. 초기화 시점에 SharedPreferences를 동기적으로 접근

**AI가 생성하는 코드:**

```dart
// ❌ 위젯 build()에서 직접 SharedPreferences 접근
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // build()에서 비동기 호출 — 매 빌드마다 실행됨
    final prefs = SharedPreferencesAsync();
    return FutureBuilder(
      future: prefs.getString('theme'), // 매 build마다 새로운 Future 생성
      builder: (context, snapshot) {
        // ...
      },
    );
  }
}
```

**무엇이 잘못되었는가:** `build()`에서 매번 `Future`를 생성하면 위젯이 리빌드될 때마다 비동기 호출이 반복되고, FutureBuilder가 매번 초기 상태를 거쳐 UI가 깜빡입니다.

**올바른 접근법:**

```dart
// ✅ 앱 시작 시 Repository를 통해 초기값 로드 → 상태 관리로 전달
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  configureDependencies(); // DI 설정

  final settingsRepo = getIt<AppSettingsRepository>();
  final initialTheme = await settingsRepo.getThemeMode();

  runApp(MyApp(initialTheme: initialTheme));
}
```

## 참고 자료

- [shared_preferences 패키지](https://pub.dev/packages/shared_preferences) — 공식 패키지 (Flutter Favorite)
- [SharedPreferencesAsync 마이그레이션 가이드](https://pub.dev/packages/shared_preferences#migrating-from-sharedpreferences-to-sharedpreferencesasync) — 레거시 API에서 마이그레이션
