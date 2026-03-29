# Result Pattern (Monadic Error Handling)

## Overview

`Result<T>`는 Dart의 `sealed class`를 활용한 모나드식 예외 처리 패턴이다. 함수의 성공(`Ok`)과 실패(`Error`)를 타입 시스템으로 명시적으로 표현하여, try-catch에 의존하지 않고 컴파일 타임에 에러 처리를 강제한다. Railway-oriented programming의 Dart 구현체로, Repository → UseCase → ViewModel 전 레이어에서 일관된 에러 전파가 가능하다.

## When To Use

- Repository 레이어에서 외부 API/DB 호출 결과를 반환할 때
- UseCase에서 비즈니스 로직의 성공/실패를 명시적으로 표현할 때
- 여러 비동기 작업을 순차적으로 체이닝하며 에러를 전파할 때 (`flatMap`)
- ViewModel에서 UI 상태 분기를 패턴 매칭으로 처리할 때

## When NOT To Use

- 절대 실패하지 않는 순수 함수 → 그냥 `T`를 반환
- Stream 기반 실시간 데이터 → `Stream<Result<T>>` 대신 Stream 자체의 `onError` 활용
- 단순 null 체크만 필요한 경우 → `T?` nullable 타입 사용
- Flutter 프레임워크가 예외를 기대하는 경우 (예: `Future.error`를 받는 FutureBuilder) → try-catch 유지

## Required Implementation

### Core: sealed class Result

```dart
sealed class Result<T> {
  const Result();

  const factory Result.ok(T value) = Ok._;
  const factory Result.error(CustomException error) = Error._;
}

final class Ok<T> extends Result<T> {
  const Ok._(this.value);
  final T value;
}

final class Error<T> extends Result<T> {
  const Error._(this.error);
  final CustomException error;
}
```

`sealed class`를 사용하므로 `Ok`와 `Error` 외의 서브타입이 존재할 수 없고, switch 패턴 매칭에서 exhaustive check가 보장된다.

### 편의 getter

```dart
// Result 내부
bool get isOk => this is Ok<T>;
bool get isError => this is Error<T>;

Ok<T> get asOk => this as Ok<T>;
Error<T> get asError => this as Error<T>;
```

### when / asyncWhen — 사이드 이펙트 처리

```dart
// 동기 콜백
result.when(
  ok: (user) => print('Hello, ${user.name}'),
  error: (e) => print('Failed: ${e.message}'),
);

// 비동기 콜백
await result.asyncWhen(
  ok: (user) async => await cacheUser(user),
  error: (e) async => await logError(e),
);
```

### map — 값 변환

```dart
// Result<User> → String 변환
final greeting = result.map<String>(
  ok: (user) => 'Hello, ${user.name}',
  error: (e) => 'Error: ${e.message}',
);
```

### flatMap / asyncFlatMap — 모나드 체이닝

```dart
// 동기 체이닝: 실패 시 자동으로 Error 전파
Result<User> getUser(String id);
Result<Profile> getProfile(User user);

final result = getUser('123').flatMap((user) => getProfile(user));

// 비동기 체이닝
Future<Result<User>> getUser(String id);
Future<Result<Profile>> getProfile(User user);

final result = await getUser('123').asyncFlatMap((user) => getProfile(user));
```

### getOrElse / getOrThrow — 값 추출

```dart
// 기본값 제공
final name = result.getOrElse(User.anonymous);

// 에러 시 예외 throw (확실히 Ok일 때만 사용)
try {
  final user = result.getOrThrow();
} catch (e) {
  // CustomException이 throw됨
}
```

### mapError — 에러 변환

```dart
// 하위 레이어 에러를 상위 도메인 에러로 변환
final result = fetchData().mapError(
  (e) => CustomNetworkException('Network failed: ${e.message}'),
);
```

### Key Rules

1. **Repository의 모든 public 메서드는 `Future<Result<T>>`를 반환해야 한다** — 예외를 throw하지 않고 Result로 감싼다
2. **CustomException을 에러 타입으로 통일한다** — 일반 `Exception`이 아닌 도메인 특화 예외 클래스를 사용
3. **`Ok`과 `Error`의 생성자는 private(`._`)으로 유지한다** — `Result.ok()`, `Result.error()` 팩토리만 사용
4. **`when`은 사이드 이펙트, `map`은 값 변환에 사용한다** — 혼용하지 않는다
5. **`flatMap`으로 체이닝할 때 중간 에러는 자동 전파된다** — 명시적 에러 체크 불필요

## Architecture / Structure

```
lib/
├── core/
│   ├── result/
│   │   └── result.dart           # Result<T>, Ok<T>, Error<T>
│   └── util/
│       └── custom_exception.dart # 공통 예외 클래스 계층
├── data/
│   └── repository/
│       └── user_repository_impl.dart  # Result 반환
├── domain/
│   ├── repository/
│   │   └── user_repository.dart       # Result 반환 인터페이스
│   └── usecase/
│       └── get_user_usecase.dart      # flatMap으로 체이닝
└── presentation/
    └── viewmodel/
        └── user_viewmodel.dart        # when/map으로 UI 상태 분기
```

### 레이어별 사용 예시

```dart
// Repository — try-catch를 Result로 변환하는 유일한 레이어
class UserRepositoryImpl implements UserRepository {
  @override
  Future<Result<User>> getUser(String id) async {
    try {
      final response = await api.fetchUser(id);
      return Result.ok(User.fromJson(response));
    } on DioException catch (e) {
      return Result.error(NetworkException(e.message ?? 'Network error'));
    } catch (e) {
      return Result.error(UnknownException(e.toString()));
    }
  }
}

// UseCase — flatMap으로 비즈니스 로직 체이닝
class GetUserProfileUseCase {
  final UserRepository _userRepo;
  final ProfileRepository _profileRepo;

  Future<Result<Profile>> call(String userId) async {
    final userResult = await _userRepo.getUser(userId);
    return userResult.asyncFlatMap(
      (user) => _profileRepo.getProfile(user.id),
    );
  }
}

// ViewModel — when으로 UI 상태 업데이트
class UserViewModel extends ChangeNotifier {
  Future<void> loadUser(String id) async {
    state = const UserState.loading();
    notifyListeners();

    final result = await _getUserProfileUseCase(id);
    result.when(
      ok: (profile) => state = UserState.loaded(profile),
      error: (e) => state = UserState.error(e.message),
    );
    notifyListeners();
  }
}
```

## Anti-Patterns

### 1. Result 안에서 try-catch 재사용

**What it looks like:**

```dart
final result = await getUser(id);
try {
  final user = result.getOrThrow();
  // user 사용
} catch (e) {
  // 에러 처리
}
```

**Why it's wrong:** Result의 존재 이유를 무시하고 다시 try-catch로 돌아간다.

**Do this instead:**

```dart
final result = await getUser(id);
result.when(
  ok: (user) {
    // user 사용
  },
  error: (e) {
    // 에러 처리
  },
);
```

### 2. Result를 무시하고 Ok만 가정

**What it looks like:**

```dart
final result = await getUser(id);
final user = result.asOk.value; // Error일 때 런타임 크래시
```

**Why it's wrong:** `as` 캐스팅은 Error일 때 `TypeError`를 발생시켜 Result 패턴의 타입 안전성을 우회한다.

**Do this instead:**

```dart
final result = await getUser(id);
final userName = result.map(
  ok: (user) => user.name,
  error: (e) => 'Unknown',
);
```

### 3. Repository 외부에서 try-catch를 Result로 감싸기

**What it looks like:**

```dart
// UseCase에서 try-catch
class GetUserUseCase {
  Future<Result<User>> call(String id) async {
    try {
      final result = await repo.getUser(id);
      return result;
    } catch (e) {
      return Result.error(UnknownException(e.toString()));
    }
  }
}
```

**Why it's wrong:** Repository가 이미 `Result`를 반환하므로 UseCase에서 다시 try-catch를 감쌀 필요가 없다.

**Do this instead:**

```dart
class GetUserUseCase {
  Future<Result<User>> call(String id) async {
    return repo.getUser(id); // Repository가 이미 Result 반환
  }
}
```

### 4. when 콜백에서 값을 반환하려고 시도

**What it looks like:**

```dart
String name = '';
result.when(
  ok: (user) => name = user.name,  // 사이드 이펙트로 값 추출
  error: (e) => name = 'Unknown',
);
```

**Why it's wrong:** `when`은 `void`를 반환하므로 외부 변수에 할당하는 것은 `map`의 역할이다.

**Do this instead:**

```dart
final name = result.map(
  ok: (user) => user.name,
  error: (e) => 'Unknown',
);
```

## Decision Criteria

| Criteria | Result Pattern | try-catch | Either (dartz/fpdart) |
|---|---|---|---|
| 타입 안전성 | sealed class exhaustive check | 런타임에만 확인 | sealed class (Left/Right) |
| 의존성 | 없음 (직접 구현) | 없음 (언어 내장) | 외부 패키지 필요 |
| 러닝 커브 | 낮음 | 매우 낮음 | 높음 (FP 개념 필요) |
| 체이닝 | flatMap/asyncFlatMap | 불가 | bind/flatMap + 풍부한 연산자 |
| 에러 타입 강제 | CustomException 고정 | 아무 타입이나 throw 가능 | 제네릭 (Left 타입 자유) |
| Dart 관용성 | 높음 (sealed class는 Dart 3 기능) | 높음 | 낮음 (Haskell/Scala 스타일) |

**Decision rule:** 외부 패키지 없이 타입 안전한 에러 처리가 필요하면 Result Pattern을 사용한다. FP 연산자(traverse, sequence 등)가 필요하면 fpdart의 Either를 사용한다. 단순한 유틸 함수나 프레임워크가 예외를 기대하는 곳에서는 try-catch를 유지한다.

## Testing Strategy

```dart
// Result를 반환하는 Repository 테스트
void main() {
  late UserRepositoryImpl repository;
  late MockApiClient mockApi;

  setUp(() {
    mockApi = MockApiClient();
    repository = UserRepositoryImpl(api: mockApi);
  });

  test('성공 시 Result.ok를 반환한다', () async {
    when(() => mockApi.fetchUser('1'))
        .thenAnswer((_) async => {'id': '1', 'name': 'Alice'});

    final result = await repository.getUser('1');

    expect(result.isOk, isTrue);
    expect(result.asOk.value.name, equals('Alice'));
  });

  test('네트워크 에러 시 Result.error를 반환한다', () async {
    when(() => mockApi.fetchUser('1'))
        .thenThrow(DioException(requestOptions: RequestOptions()));

    final result = await repository.getUser('1');

    expect(result.isError, isTrue);
    expect(result.asError.error, isA<NetworkException>());
  });

  test('flatMap 체이닝에서 첫 번째 실패 시 두 번째는 실행되지 않는다', () async {
    final error = Result<User>.error(NetworkException('fail'));
    var secondCalled = false;

    final result = await error.asyncFlatMap((user) async {
      secondCalled = true;
      return Result.ok(Profile(user: user));
    });

    expect(result.isError, isTrue);
    expect(secondCalled, isFalse);
  });

  test('map으로 값을 변환한다', () {
    final result = Result.ok(User(name: 'Alice'));

    final name = result.map(
      ok: (user) => user.name,
      error: (e) => 'Unknown',
    );

    expect(name, equals('Alice'));
  });
}
```

### What to Mock

- API 클라이언트 / 데이터 소스 — Repository가 try-catch → Result 변환을 올바르게 하는지 검증
- Repository (UseCase 테스트 시) — `Result.ok()` 또는 `Result.error()`를 직접 반환하도록 stub

### What NOT to Mock

- `Result` 자체 — sealed class이므로 실제 `Result.ok()` / `Result.error()`를 사용
- `flatMap`, `when`, `map` 등 Result 메서드 — 순수 함수이므로 실제 동작으로 검증

## Common AI Mistakes

### 1. sealed class 대신 abstract class 사용

**What AI generates:**

```dart
abstract class Result<T> {
  const Result();
}

class Ok<T> extends Result<T> {
  final T value;
  const Ok(this.value);
}

class Error<T> extends Result<T> {
  final Exception error;
  const Error(this.error);
}
```

**What's wrong:** `abstract class`는 외부 서브클래스를 허용하므로 switch에서 exhaustive 패턴 매칭이 불가능하다.

**Correct approach:**

```dart
sealed class Result<T> {
  const Result();
  const factory Result.ok(T value) = Ok._;
  const factory Result.error(CustomException error) = Error._;
}

// switch 패턴 매칭에서 exhaustive check 보장
final message = switch (result) {
  Ok(value: final user) => 'Hello, ${user.name}',
  Error(error: final e) => 'Error: ${e.message}',
};
```

### 2. 에러 타입을 generic Exception으로 사용

**What AI generates:**

```dart
sealed class Result<T> {
  const factory Result.ok(T value) = Ok._;
  const factory Result.error(Exception error) = Error._; // 일반 Exception
}
```

**What's wrong:** `Exception`은 너무 광범위하여 에러 분기 처리 시 `is` 체크의 연쇄가 필요해진다.

**Correct approach:**

```dart
// 도메인 예외 계층 정의
sealed class CustomException implements Exception {
  final String message;
  const CustomException(this.message);
}

class NetworkException extends CustomException {
  const NetworkException(super.message);
}

class ValidationException extends CustomException {
  final Map<String, String> fieldErrors;
  const ValidationException(super.message, {this.fieldErrors = const {}});
}

// Result에서 CustomException 사용
sealed class Result<T> {
  const factory Result.error(CustomException error) = Error._;
}
```

### 3. flatMap 대신 중첩 when으로 체이닝

**What AI generates:**

```dart
Future<Result<Profile>> getUserProfile(String id) async {
  final userResult = await userRepo.getUser(id);
  return userResult.map(
    ok: (user) async {
      final profileResult = await profileRepo.getProfile(user.id);
      return profileResult.map(
        ok: (profile) => Result.ok(profile),
        error: (e) => Result.error(e),
      );
    },
    error: (e) => Result.error(e),
  );
  // 반환 타입이 Future<dynamic>이 되어 타입 에러 발생
}
```

**What's wrong:** `map` 중첩은 비동기 처리 시 타입 추론이 깨지며, 이것이 `flatMap`이 존재하는 이유다.

**Correct approach:**

```dart
Future<Result<Profile>> getUserProfile(String id) async {
  final userResult = await userRepo.getUser(id);
  return userResult.asyncFlatMap(
    (user) => profileRepo.getProfile(user.id),
  );
}
```

## References

- [Dart Sealed Classes](https://dart.dev/language/class-modifiers#sealed) — sealed class 공식 문서
- [Dart Patterns](https://dart.dev/language/patterns) — 패턴 매칭 공식 문서
- [Andrea Bizzotto - Result Type](https://codewithandrea.com/articles/flutter-exception-handling-try-catch-result-type/) — Result 패턴 해설
