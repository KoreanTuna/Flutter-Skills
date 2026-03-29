# get_it + injectable을 활용한 의존성 주입

## 개요

`get_it` Service Locator와 `injectable` 코드 생성기를 조합하여 Flutter 앱의 의존성 주입(DI)을 구성하는 패턴을 정의합니다. `get_it`이 런타임 DI 컨테이너 역할을 하고, `injectable`이 등록 보일러플레이트를 어노테이션 기반으로 자동 생성하여 수동 등록 실수를 방지합니다.

## 사용 시점

- Repository, DataSource, UseCase 등 계층 간 의존성을 느슨하게 결합해야 할 때
- 테스트에서 실제 구현을 Mock으로 교체해야 할 때
- 앱 규모가 커져 수동 `getIt.registerFactory(...)` 등록이 관리 불가능해질 때
- 환경별(dev/staging/prod) 의존성을 다르게 구성해야 할 때

## 사용하지 말아야 할 때

- 단순히 상수나 설정값을 공유하는 경우 → `const` 값이나 `Environment` 클래스로 충분

## 필수 구현

### 1. 패키지 설정

```yaml
# pubspec.yaml
dependencies:
  get_it: ^8.0.3
  injectable: ^2.5.0

dev_dependencies:
  injectable_generator: ^2.7.0
  build_runner: ^2.4.14
```

### 2. Service Locator 초기화

```dart
// lib/core/di/injection.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection.config.dart'; // 코드 생성 파일

final getIt = GetIt.instance; // 전역 접근점 — 앱 전체에서 단일 인스턴스

@InjectableInit() // injectable이 이 함수 내부에 등록 코드를 생성
void configureDependencies(String environment) =>
    getIt.init(environment: environment);
```

### 3. main에서 초기화 호출

```dart
// lib/main.dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  configureDependencies(Environment.prod); // 환경 지정
  runApp(const MyApp());
}
```

### 4. 의존성 등록 — 어노테이션

```dart
// 인터페이스 정의
abstract class AuthRepository {
  Future<User> getCurrentUser();
  Future<void> signOut();
}

// 구현체 등록 — @Injectable로 자동 등록
@Injectable(as: AuthRepository) // 인터페이스에 대한 구현체로 등록
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remoteDataSource;
  final AuthLocalDataSource _localDataSource;

  // 생성자 파라미터가 자동으로 의존성 그래프에 포함됨
  AuthRepositoryImpl(this._remoteDataSource, this._localDataSource);

  @override
  Future<User> getCurrentUser() => _remoteDataSource.fetchCurrentUser();

  @override
  Future<void> signOut() => _localDataSource.clearToken();
}
```

### 5. 등록 유형별 어노테이션

```dart
// Singleton — 앱 수명 동안 단일 인스턴스
@singleton
class AppRouter {
  final GoRouter router;
  AppRouter(this.router);
}

// LazySingleton — 첫 접근 시 생성, 이후 동일 인스턴스
@lazySingleton
class DatabaseService {
  Database? _db;

  Future<Database> get database async {
    _db ??= await openDatabase('app.db');
    return _db!;
  }
}

// Injectable (Factory) — 매번 새 인스턴스 생성
@injectable
class FetchUserUseCase {
  final AuthRepository _authRepository;
  FetchUserUseCase(this._authRepository);
}
```

### 6. 의존성 사용

```dart
// Widget에서 사용 — BLoC/Cubit 주입 시
class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => getIt<HomeCubit>(), // Service Locator로 가져오기
      child: const HomeView(),
    );
  }
}
```

### 7. 환경별 구성

```dart
// 개발 환경 전용 구현체
@dev // Environment.dev에서만 등록
@Injectable(as: AuthRepository)
class FakeAuthRepository implements AuthRepository {
  @override
  Future<User> getCurrentUser() async => const User(id: 'fake', name: 'Dev User');

  @override
  Future<void> signOut() async {}
}

// 프로덕션 환경 전용 구현체
@prod // Environment.prod에서만 등록
@Injectable(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remoteDataSource;
  final AuthLocalDataSource _localDataSource;
  AuthRepositoryImpl(this._remoteDataSource, this._localDataSource);

  @override
  Future<User> getCurrentUser() => _remoteDataSource.fetchCurrentUser();

  @override
  Future<void> signOut() => _localDataSource.clearToken();
}

// 커스텀 환경 정의
const staging = Environment('staging');

@staging
@Injectable(as: AnalyticsService)
class StagingAnalyticsService implements AnalyticsService {
  @override
  void logEvent(String name) => debugPrint('[STAGING] $name');
}
```

### 8. 외부 패키지 의존성 등록 — @module

```dart
// lib/core/di/modules/network_module.dart
@module // injectable이 이 클래스의 메서드를 등록 코드로 변환
abstract class NetworkModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: const Duration(seconds: 10),
  ));

  @lazySingleton
  SharedPreferences get prefs => throw UnimplementedError();

  // 비동기 초기화가 필요한 의존성
  @preResolve // init 시 await로 미리 생성
  @lazySingleton
  Future<SharedPreferences> get sharedPreferences =>
      SharedPreferences.getInstance();
}
```

비동기 의존성(`@preResolve`) 사용 시 초기화 함수를 `async`로 변경:

```dart
// lib/core/di/injection.dart
@InjectableInit()
Future<void> configureDependencies(String environment) async =>
    await getIt.init(environment: environment); // await 필수
```

### 9. 코드 생성 실행

```bash
dart run build_runner build --delete-conflicting-outputs
```

### 핵심 규칙

1. **인터페이스 기반 등록 필수** — 구현체를 직접 등록하지 않고 `@Injectable(as: Interface)` 형태로 등록하여 교체 가능성을 확보
2. **생성자 주입만 사용** — 필드 주입, setter 주입 금지. injectable이 생성자 파라미터를 분석하여 의존성 그래프를 자동 구성하므로 생성자 외 주입은 그래프에 잡히지 않음
3. **getIt 직접 호출은 Composition Root에서만** — Widget의 `build`나 BLoC `create`에서만 `getIt<T>()`를 호출. 비즈니스 로직 내부에서 Service Locator 패턴으로 호출하면 숨은 의존성이 생김
4. **순환 의존성 금지** — A → B → A 형태의 의존은 injectable 코드 생성 시 에러 발생. 인터페이스 분리 또는 설계 변경으로 해결
5. **`build_runner` 실행 후 생성 파일 커밋** — `injection.config.dart`는 생성 파일이지만 CI에서 build_runner 실행을 보장할 수 없다면 커밋에 포함

## 아키텍처 / 구조

```
lib/
├── core/
│   └── di/
│       ├── injection.dart           # getIt 인스턴스 + configureDependencies
│       ├── injection.config.dart    # 자동 생성 파일
│       └── modules/                 # @module 클래스 (외부 패키지 등록)
│           ├── network_module.dart
│           └── storage_module.dart
├── feature/
│   └── auth/
│       ├── domain/
│       │   ├── repository/
│       │   │   └── auth_repository.dart       # 인터페이스
│       │   └── usecase/
│       │       └── fetch_user_use_case.dart   # @injectable
│       ├── data/
│       │   ├── repository/
│       │   │   └── auth_repository_impl.dart  # @Injectable(as: AuthRepository)
│       │   └── data_source/
│       │       ├── auth_remote_data_source.dart
│       │       └── auth_local_data_source.dart
│       └── presentation/
│           ├── cubit/
│           │   └── auth_cubit.dart            # @injectable
│           └── view/
│               └── login_view.dart
└── main.dart
```

- `@module` 클래스는 `core/di/modules/`에 집중 배치
- 각 feature의 구현체에 직접 어노테이션을 부착 — 별도 등록 파일 불필요
- 인터페이스와 구현체는 반드시 다른 디렉토리에 위치 (domain vs data)

## 안티패턴

### 1. 비즈니스 로직 내부에서 getIt 직접 호출

**잘못된 코드:**

```dart
class FetchOrdersUseCase {
  // getIt을 내부에서 직접 호출 — 숨은 의존성
  Future<List<Order>> call() {
    final repository = getIt<OrderRepository>();
    return repository.fetchOrders();
  }
}
```

**왜 잘못되었는가:** 의존성이 생성자에 드러나지 않아 테스트 시 Mock 주입이 불가능합니다.

**올바른 방법:**

```dart
@injectable
class FetchOrdersUseCase {
  final OrderRepository _repository; // 생성자 주입
  FetchOrdersUseCase(this._repository);

  Future<List<Order>> call() => _repository.fetchOrders();
}
```

### 2. 구현체를 인터페이스 없이 직접 등록

**잘못된 코드:**

```dart
@lazySingleton // 구현체를 바로 등록
class AuthRepositoryImpl {
  final Dio _dio;
  AuthRepositoryImpl(this._dio);
}

// 사용처에서 구현체에 직접 의존
@injectable
class LoginUseCase {
  final AuthRepositoryImpl _repo; // 구현체 타입
  LoginUseCase(this._repo);
}
```

**왜 잘못되었는가:** 테스트에서 Mock으로 교체할 수 없고 구현체 변경 시 모든 사용처를 수정해야 합니다.

**올바른 방법:**

```dart
abstract class AuthRepository {
  Future<User> login(String email, String password);
}

@Injectable(as: AuthRepository) // 인터페이스로 등록
class AuthRepositoryImpl implements AuthRepository {
  final Dio _dio;
  AuthRepositoryImpl(this._dio);

  @override
  Future<User> login(String email, String password) async {
    final response = await _dio.post('/login', data: {'email': email, 'password': password});
    return User.fromJson(response.data);
  }
}

@injectable
class LoginUseCase {
  final AuthRepository _repo; // 인터페이스 타입
  LoginUseCase(this._repo);
}
```

### 3. 모든 것을 Singleton으로 등록

**잘못된 코드:**

```dart
@singleton // UseCase를 Singleton으로 — 불필요한 메모리 점유
class FetchUserUseCase {
  final AuthRepository _repo;
  FetchUserUseCase(this._repo);
}

@singleton // Cubit을 Singleton으로 — 상태 누수
class HomeCubit extends Cubit<HomeState> {
  HomeCubit(this._fetchUserUseCase) : super(const HomeState.initial());
  final FetchUserUseCase _fetchUserUseCase;
}
```

**왜 잘못되었는가:** Singleton Cubit은 화면 간 상태가 공유되어 이전 화면의 stale 상태가 남아있게 됩니다.

**올바른 방법:**

```dart
@injectable // Factory — 매번 새 인스턴스
class FetchUserUseCase {
  final AuthRepository _repo;
  FetchUserUseCase(this._repo);
}

@injectable // Factory — 화면 진입 시마다 새로운 상태
class HomeCubit extends Cubit<HomeState> {
  HomeCubit(this._fetchUserUseCase) : super(const HomeState.initial());
  final FetchUserUseCase _fetchUserUseCase;
}
```

**등록 유형 기준:**

| 유형                | Singleton / LazySingleton | Factory (@injectable) |
| ------------------- | ------------------------- | --------------------- |
| DataSource, Dio, DB | O                         | X                     |
| Repository          | O (LazySingleton)         | X                     |
| UseCase             | X                         | O                     |
| BLoC / Cubit        | X                         | O                     |
| Router, AppConfig   | O (Singleton)             | X                     |

### 4. @module에서 구현 로직 작성

**잘못된 코드:**

```dart
@module
abstract class AuthModule {
  @injectable
  AuthRepository get authRepository => AuthRepositoryImpl(
    AuthRemoteDataSource(getIt<Dio>()), // module 내부에서 getIt 호출 + 구현 로직
    AuthLocalDataSource(getIt<SharedPreferences>()),
  );
}
```

**왜 잘못되었는가:** `@module`은 어노테이션을 부착할 수 없는 외부 패키지 전용이며, 직접 제어 가능한 클래스는 클래스에 어노테이션을 부착해야 합니다.

**올바른 방법:**

```dart
// 각 클래스에 직접 어노테이션 부착
@Injectable(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  AuthRepositoryImpl(this._remoteDataSource, this._localDataSource);
  final AuthRemoteDataSource _remoteDataSource;
  final AuthLocalDataSource _localDataSource;
}

@injectable
class AuthRemoteDataSource {
  AuthRemoteDataSource(this._dio);
  final Dio _dio;
}

// @module은 Dio처럼 직접 수정할 수 없는 것만
@module
abstract class NetworkModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(baseUrl: 'https://api.example.com'));
}
```

## 의사결정 기준

### DI 접근법 선택

| 기준               | get_it + injectable                      | Riverpod                  | 수동 생성자 주입 |
| ------------------ | ---------------------------------------- | ------------------------- | ---------------- |
| 보일러플레이트     | 어노테이션 + 코드 생성                   | Provider 선언             | 없음             |
| 학습 비용          | 중간                                     | 높음                      | 낮음             |
| Widget 트리 의존성 | 없음 (Service Locator)                   | 있음 (ProviderScope)      | 있음             |
| 런타임 교체        | `getIt.unregister` / `registerSingleton` | Provider override         | 불가             |
| 테스트 용이성      | 높음                                     | 높음                      | 중간             |
| 코드 생성 필요     | 필수 (build_runner)                      | 선택 (riverpod_generator) | 불필요           |
| 적합한 규모        | 중~대규모                                | 중~대규모                 | 소규모           |

**의사결정 규칙:** 의존성 5개 미만이면 수동 생성자 주입. Riverpod을 이미 사용 중이면 Riverpod의 DI 기능 활용. BLoC/Clean Architecture 기반 프로젝트에서 계층 간 DI가 필요하면 get_it + injectable.

## 테스트 전략

### 테스트용 DI 구성

```dart
// test/helpers/test_injection.dart
import 'package:get_it/get_it.dart';

void setupTestDependencies() {
  final getIt = GetIt.instance;
  getIt.reset(); // 이전 테스트 등록 초기화

  // Mock 등록
  getIt.registerFactory<AuthRepository>(() => MockAuthRepository());
  getIt.registerFactory<HomeCubit>(
    () => HomeCubit(getIt<AuthRepository>() as MockAuthRepository),
  );
}

void tearDownTestDependencies() {
  GetIt.instance.reset();
}
```

### UseCase 단위 테스트

```dart
void main() {
  late MockAuthRepository mockAuthRepository;
  late FetchUserUseCase sut;

  setUp(() {
    mockAuthRepository = MockAuthRepository();
    sut = FetchUserUseCase(mockAuthRepository); // 생성자 주입 — getIt 불필요
  });

  test('getCurrentUser returns user from repository', () async {
    const expected = User(id: '1', name: 'Test');
    when(() => mockAuthRepository.getCurrentUser())
        .thenAnswer((_) async => expected);

    final result = await sut.call();

    expect(result, expected);
    verify(() => mockAuthRepository.getCurrentUser()).called(1);
  });
}
```

### Widget 테스트 — getIt Mock 교체

```dart
void main() {
  setUp(() => setupTestDependencies());
  tearDown(() => tearDownTestDependencies());

  testWidgets('HomePage renders user name', (tester) async {
    final mockCubit = getIt<HomeCubit>();

    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<HomeCubit>.value(
          value: mockCubit,
          child: const HomeView(),
        ),
      ),
    );

    // Cubit 상태 변경 후 검증
    expect(find.text('Test User'), findsOneWidget);
  });
}
```

### Mock 해야 하는 것

- Repository — 외부 API, DB 접근을 격리
- DataSource — 네트워크, 파일시스템 의존성 제거
- 환경별 서비스 (Analytics, Crash Reporting 등)

### Mock 하지 말아야 하는 것

- UseCase — 비즈니스 로직 자체를 테스트해야 하므로 실제 인스턴스 사용
- 값 객체 (Entity, DTO) — 불변 데이터 클래스는 그대로 사용
- get_it 자체 — 단위 테스트에서는 생성자 주입으로 직접 전달, Widget 테스트에서만 getIt.reset + 재등록

## AI가 자주 하는 실수

### 1. injectable 어노테이션 없이 수동 등록 코드 작성

**AI가 생성하는 코드:**

```dart
// injection.dart — 수동으로 모든 의존성 등록
void configureDependencies() {
  getIt.registerLazySingleton<Dio>(() => Dio());
  getIt.registerLazySingleton<AuthRepository>(
    () => AuthRepositoryImpl(getIt<Dio>()),
  );
  getIt.registerFactory<AuthCubit>(
    () => AuthCubit(getIt<AuthRepository>()),
  );
  // ... 의존성이 추가될 때마다 수동 등록 필요
}
```

**무엇이 잘못되었는가:** injectable 프로젝트에서 수동 등록은 코드 생성과 충돌하며, injectable 도입 목적에 반합니다.

**올바른 접근법:**

```dart
// 각 클래스에 어노테이션만 부착
@lazySingleton
class AuthRemoteDataSource {
  AuthRemoteDataSource(this._dio);
  final Dio _dio;
}

@Injectable(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  AuthRepositoryImpl(this._remoteDataSource);
  final AuthRemoteDataSource _remoteDataSource;
}

@injectable
class AuthCubit extends Cubit<AuthState> {
  AuthCubit(this._authRepository) : super(const AuthState.initial());
  final AuthRepository _authRepository;
}

// Dio만 @module로 등록 (외부 패키지)
@module
abstract class NetworkModule {
  @lazySingleton
  Dio get dio => Dio();
}
```

### 2. Cubit/BLoC을 Singleton으로 등록

**AI가 생성하는 코드:**

```dart
@lazySingleton // AI가 "앱 전역에서 사용"이라는 이유로 Singleton 적용
class CartCubit extends Cubit<CartState> {
  CartCubit(this._cartRepository) : super(const CartState.initial());
  final CartRepository _cartRepository;
}
```

**무엇이 잘못되었는가:** Singleton Cubit은 `close()`가 호출되지 않아 Stream 누수가 발생하고, 화면 재진입 시 stale 상태가 표시됩니다.

**올바른 접근법:**

```dart
@injectable // Factory — BlocProvider가 생성/해제 생명주기를 관리
class CartCubit extends Cubit<CartState> {
  CartCubit(this._cartRepository) : super(const CartState.initial());
  final CartRepository _cartRepository;
}

// 진짜 앱 전역 상태가 필요하면 별도 Singleton 서비스로 분리
@lazySingleton
class CartService {
  final CartRepository _cartRepository;
  CartService(this._cartRepository);

  final _cartStream = BehaviorSubject<Cart>.seeded(const Cart.empty());
  Stream<Cart> get cartStream => _cartStream.stream;
}
```

### 3. 생성된 파일(injection.config.dart)을 직접 수정

**AI가 생성하는 코드:**

```dart
// injection.config.dart — AI가 직접 편집
@InjectableInit()
GetIt init(GetIt getIt, {String? environment}) {
  getIt.registerFactory<AuthCubit>(() => AuthCubit(getIt<AuthRepository>()));
  getIt.registerFactory<NewFeatureCubit>(() => NewFeatureCubit()); // AI가 직접 추가
  return getIt;
}
```

**무엇이 잘못되었는가:** `injection.config.dart`는 `build_runner`가 생성하는 파일이므로 직접 수정하면 다음 생성 시 덮어씌워집니다.

**올바른 접근법:**

```dart
// 새 클래스에 어노테이션을 붙이고
@injectable
class NewFeatureCubit extends Cubit<NewFeatureState> {
  NewFeatureCubit() : super(const NewFeatureState.initial());
}

// build_runner를 다시 실행
// dart run build_runner build --delete-conflicting-outputs
```

## 참고 자료

- [get_it 패키지](https://pub.dev/packages/get_it) — Service Locator 기본 API 및 등록 유형
- [injectable 패키지](https://pub.dev/packages/injectable) — 어노테이션 목록 및 코드 생성 규칙
- [injectable GitHub](https://github.com/Milad-Akarie/injectable) — 환경 설정, @module, @preResolve 등 고급 사용법
