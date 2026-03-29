# 데이터 모델링

## 개요

Flutter 프로젝트에서 Domain Model, DTO, Enum, 결과 객체 등 데이터 클래스를 설계하고 관리하는 패턴을 정의합니다. Freezed를 활용한 불변 객체 생성, Domain과 Data 레이어 간 모델 분리, Mapper를 통한 변환, sealed class를 활용한 union type 패턴을 다룹니다.

## 사용 시점

- 새로운 Feature의 도메인 모델과 DTO를 설계할 때
- API 응답/요청 객체와 비즈니스 모델을 분리해야 할 때
- 상태 표현에 union type(sealed class)이 필요할 때
- 기존 모델을 리팩토링하여 레이어 간 경계를 명확히 해야 할 때

## 사용하지 말아야 할 때

- 단순 UI 전용 설정값(탭 인덱스, 토글 등) → `useState` 또는 `ValueNotifier` 사용
- 외부 패키지가 정의한 모델을 그대로 사용하는 경우 → 래핑 없이 직접 사용
- 1~2개 필드의 임시 데이터 전달 → Record `(String, int)` 또는 named parameter로 충분

## 필수 구현

### Domain Model — 순수 Freezed

```dart
// domain/model/user.dart
@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    required DateTime createdAt,     // String이 아닌 DateTime
    required UserRole role,          // String이 아닌 Enum
    @Default(false) bool isVerified, // 기본값은 @Default로
  }) = _User;
}
```

**Domain Model 규칙:**
- `@JsonSerializable`, `@JsonKey`, `fromJson`/`toJson` 금지 — JSON 직렬화는 DTO의 책임
- 원시 타입(`String`) 대신 의미 있는 타입(`DateTime`, `Enum`) 사용
- nullable은 비즈니스적으로 "없을 수 있는" 경우에만 사용

### DTO — Freezed + JsonSerializable

```dart
// data/dto/response/user_response_dto.dart
@freezed
class UserResponseDto with _$UserResponseDto {
  const factory UserResponseDto({
    required String id,
    required String name,
    required String email,
    @JsonKey(name: 'created_at') required String createdAt, // API 필드명 매핑
    required String role,
    @JsonKey(name: 'is_verified') @Default(false) bool isVerified,
  }) = _UserResponseDto;

  factory UserResponseDto.fromJson(Map<String, dynamic> json) =>
      _$UserResponseDtoFromJson(json);
}
```

```dart
// data/dto/request/create_user_request_dto.dart
@freezed
class CreateUserRequestDto with _$CreateUserRequestDto {
  const factory CreateUserRequestDto({
    required String name,
    required String email,
    required String role,
  }) = _CreateUserRequestDto;

  factory CreateUserRequestDto.fromJson(Map<String, dynamic> json) =>
      _$CreateUserRequestDtoFromJson(json);
}
```

### Mapper — DTO ↔ Domain 변환

```dart
// data/mapper/user_mapper.dart
@injectable
class UserMapper {
  User fromDto(UserResponseDto dto) {
    return User(
      id: dto.id,
      name: dto.name,
      email: dto.email,
      createdAt: DateTime.parse(dto.createdAt), // String → DateTime 변환
      role: UserRole.fromString(dto.role),       // String → Enum 변환
      isVerified: dto.isVerified,
    );
  }

  CreateUserRequestDto toDto(User user) {
    return CreateUserRequestDto(
      name: user.name,
      email: user.email,
      role: user.role.name,
    );
  }
}
```

### Enum — 도메인 열거형

```dart
// domain/enum/user_role.dart
enum UserRole {
  admin,
  member,
  guest;

  // API 문자열 → Enum 변환 (알 수 없는 값 대응)
  static UserRole fromString(String value) {
    return UserRole.values.firstWhere(
      (e) => e.name == value,
      orElse: () => UserRole.guest, // 알 수 없는 값은 기본값
    );
  }
}
```

### Result 객체 — 명령 결과 표현

```dart
// domain/result/create_user_result.dart
sealed class CreateUserResult {
  const CreateUserResult();
}

class CreateUserSuccess extends CreateUserResult {
  final User user;
  const CreateUserSuccess(this.user);
}

class CreateUserEmailDuplicate extends CreateUserResult {
  const CreateUserEmailDuplicate();
}

class CreateUserFailure extends CreateUserResult {
  final String message;
  const CreateUserFailure(this.message);
}
```

```dart
// BLoC에서 사용
void _onSubmit(CreateUserSubmitted event, Emitter<CreateUserState> emit) async {
  final result = await _createUserUseCase(event.user);

  switch (result) {
    case CreateUserSuccess(:final user):
      emit(state.copyWith(status: AsyncStatus.success, user: user));
    case CreateUserEmailDuplicate():
      emit(state.copyWith(status: AsyncStatus.failure, error: '이미 사용 중인 이메일'));
    case CreateUserFailure(:final message):
      emit(state.copyWith(status: AsyncStatus.failure, error: message));
  }
}
```

### Sealed Class — Union Type

```dart
// domain/model/payment_method.dart
@freezed
sealed class PaymentMethod with _$PaymentMethod {
  const factory PaymentMethod.card({
    required String cardNumber,
    required String expiryDate,
  }) = PaymentMethodCard;

  const factory PaymentMethod.bank({
    required String bankName,
    required String accountNumber,
  }) = PaymentMethodBank;

  const factory PaymentMethod.point({
    required int availablePoints,
  }) = PaymentMethodPoint;
}
```

```dart
// UI에서 switch로 분기
Widget build(BuildContext context) {
  return switch (paymentMethod) {
    PaymentMethodCard(:final cardNumber) => Text('카드: $cardNumber'),
    PaymentMethodBank(:final bankName) => Text('은행: $bankName'),
    PaymentMethodPoint(:final availablePoints) => Text('포인트: $availablePoints'),
  };
}
```

### 핵심 규칙

1. **Domain Model에 JSON 관련 코드 금지** — `@JsonSerializable`, `fromJson`, `toJson`은 DTO에서만 사용
2. **DTO와 Domain Model은 반드시 분리** — API 스펙 변경이 비즈니스 로직에 전파되지 않도록 Mapper로 변환
3. **nullable은 비즈니스 의미가 있을 때만** — "API가 null을 보내니까"가 아니라 "이 값은 비즈니스적으로 없을 수 있는가"로 판단
4. **Enum은 `fromString`에 기본값 필수** — 서버가 새 값을 추가해도 앱이 크래시하지 않도록
5. **명령 결과는 sealed class로** — `bool success` + `String? error` 대신 타입 안전한 결과 객체 사용

## 아키텍처 / 구조

```
lib/feature/{domain}/
├── data/
│   ├── dto/
│   │   ├── request/
│   │   │   └── create_user_request_dto.dart    # Freezed + JsonSerializable
│   │   └── response/
│   │       └── user_response_dto.dart          # Freezed + JsonSerializable
│   └── mapper/
│       └── user_mapper.dart                    # DTO ↔ Domain 변환
│
└── domain/
    ├── model/
    │   ├── user.dart                           # 순수 Freezed (JSON 없음)
    │   └── payment_method.dart                 # sealed class union type
    ├── result/
    │   └── create_user_result.dart             # 명령 결과 sealed class
    └── enum/
        └── user_role.dart                      # 도메인 열거형
```

- DTO는 `request/`, `response/`로 방향 구분
- Domain Model은 `model/` 하위에 평면 배치
- 복잡한 결과는 `result/`, 열거형은 `enum/`에 분리

## 안티패턴

### 1. Domain Model에 JSON 직렬화 포함

**잘못된 코드:**

```dart
// ❌ domain/model/user.dart에 JSON 관련 코드
@freezed
class User with _$User {
  const factory User({
    required String id,
    @JsonKey(name: 'user_name') required String name, // ❌ API 필드명이 도메인에 침투
    @JsonKey(name: 'created_at') required String createdAt, // ❌ String 타입
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

**왜 잘못되었는가:** API 스펙 변경이 Domain Model까지 전파되어 레이어 독립성이 깨집니다.

**올바른 방법:**

```dart
// ✅ domain/model/user.dart — 순수 Freezed
@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,          // 비즈니스 용어
    required DateTime createdAt,   // 적절한 타입
  }) = _User;
  // fromJson 없음
}

// ✅ data/dto/response/user_response_dto.dart — JSON 전담
@freezed
class UserResponseDto with _$UserResponseDto {
  const factory UserResponseDto({
    required String id,
    @JsonKey(name: 'user_name') required String name,
    @JsonKey(name: 'created_at') required String createdAt,
  }) = _UserResponseDto;

  factory UserResponseDto.fromJson(Map<String, dynamic> json) =>
      _$UserResponseDtoFromJson(json);
}
```

### 2. DTO를 UI까지 전달

**잘못된 코드:**

```dart
// ❌ BLoC State에 DTO 직접 사용
@freezed
class UserListState with _$UserListState {
  const factory UserListState({
    @Default([]) List<UserResponseDto> users, // ❌ DTO가 Presentation에 노출
  }) = _UserListState;
}
```

**왜 잘못되었는가:** API 응답 구조 변경이 State, View, Widget까지 연쇄 수정을 유발합니다.

**올바른 방법:**

```dart
// ✅ Domain Model만 State에 사용
@freezed
class UserListState with _$UserListState {
  const factory UserListState({
    @Default([]) List<User> users, // Domain Model
    @Default(AsyncStatus.initial) AsyncStatus status,
  }) = _UserListState;
}
```

### 3. 원시 타입 남용

**잘못된 코드:**

```dart
// ❌ 모든 값이 String — 타입 안전성 없음
@freezed
class Order with _$Order {
  const factory Order({
    required String id,
    required String status,        // "pending", "shipped", "delivered"?
    required String createdAt,     // "2024-01-01T00:00:00Z"?
    required String totalAmount,   // "15000"? "15,000"?
  }) = _Order;
}
```

**왜 잘못되었는가:** 오타나 잘못된 값이 컴파일 타임에 잡히지 않고 런타임 에러로 이어집니다.

**올바른 방법:**

```dart
// ✅ 의미 있는 타입 사용
@freezed
class Order with _$Order {
  const factory Order({
    required String id,
    required OrderStatus status,   // Enum
    required DateTime createdAt,   // DateTime
    required int totalAmount,      // int (원 단위)
  }) = _Order;
}
```

### 4. copyWith 대신 새 인스턴스 수동 생성

**잘못된 코드:**

```dart
// ❌ 모든 필드를 수동으로 복사
final updatedUser = User(
  id: user.id,
  name: newName,
  email: user.email,
  createdAt: user.createdAt,
  role: user.role,
  isVerified: user.isVerified,
);
```

**왜 잘못되었는가:** 필드 추가 시 복사 누락이 발생하며, Freezed의 `copyWith`를 사용하지 않는 것은 Freezed 도입 목적에 반합니다.

**올바른 방법:**

```dart
// ✅ copyWith — 변경할 필드만 명시
final updatedUser = user.copyWith(name: newName);
```

### 5. bool/String 조합으로 결과 표현

**잘못된 코드:**

```dart
// ❌ 결과를 bool + nullable String으로 표현
class CreateUserResult {
  final bool success;
  final String? errorMessage;
  final User? user;

  CreateUserResult({required this.success, this.errorMessage, this.user});
}
```

**왜 잘못되었는가:** `success == true && user == null` 같은 불가능해야 하는 상태 조합이 가능합니다.

**올바른 방법:**

```dart
// ✅ sealed class — 불가능한 상태를 컴파일 타임에 방지
sealed class CreateUserResult {
  const CreateUserResult();
}

class CreateUserSuccess extends CreateUserResult {
  final User user; // 성공이면 반드시 user 존재
  const CreateUserSuccess(this.user);
}

class CreateUserFailure extends CreateUserResult {
  final String message; // 실패면 반드시 message 존재
  const CreateUserFailure(this.message);
}
```

## 의사결정 기준

### 모델 타입 선택

| 기준 | Freezed class | Dart sealed class | Enum | Record |
|---|---|---|---|---|
| 여러 필드를 가진 데이터 | ✅ | ❌ | ❌ | 2~3개면 가능 |
| 유한한 변형(union) | ✅ Freezed union | ✅ | ❌ | ❌ |
| 유한한 고정 상수 | ❌ 과도 | ❌ 과도 | ✅ | ❌ |
| 임시 데이터 묶음 | ❌ 과도 | ❌ | ❌ | ✅ |
| copyWith 필요 | ✅ 자동 생성 | ❌ 수동 구현 | ❌ | ❌ |
| JSON 직렬화 필요 | ✅ DTO용 | ❌ | ❌ | ❌ |

**의사결정 규칙:** 여러 필드 + copyWith/equality 필요 → Freezed. 유한한 변형이 있고 switch 분기 필요 → sealed class (Freezed union 또는 순수 sealed). 고정 상수 집합 → Enum. 함수 간 임시 전달 → Record.

### nullable vs required vs @Default

| 상황 | 선택 | 이유 |
|---|---|---|
| 비즈니스적으로 반드시 존재 | `required` | null 가능성 제거 |
| 비즈니스적으로 없을 수 있음 | `T?` nullable | 프로필 사진, 별명 등 |
| 항상 존재하되 초기값 존재 | `@Default(value)` | 빈 리스트, false 등 |

## 테스트 전략

### Domain Model 테스트

```dart
test('User equality — 같은 필드면 같은 객체', () {
  final user1 = User(
    id: '1', name: 'Kim', email: 'kim@test.com',
    createdAt: DateTime(2024, 1, 1), role: UserRole.admin,
  );
  final user2 = User(
    id: '1', name: 'Kim', email: 'kim@test.com',
    createdAt: DateTime(2024, 1, 1), role: UserRole.admin,
  );

  expect(user1, equals(user2)); // Freezed가 생성한 == 연산자
});

test('User copyWith — 지정한 필드만 변경', () {
  final user = User(
    id: '1', name: 'Kim', email: 'kim@test.com',
    createdAt: DateTime(2024, 1, 1), role: UserRole.member,
  );

  final updated = user.copyWith(role: UserRole.admin);

  expect(updated.role, UserRole.admin);
  expect(updated.name, 'Kim'); // 나머지 필드 유지
});
```

### Mapper 테스트

```dart
test('UserMapper.fromDto — DTO를 Domain Model로 정확히 변환', () {
  final mapper = UserMapper();
  final dto = UserResponseDto(
    id: '1',
    name: 'Kim',
    email: 'kim@test.com',
    createdAt: '2024-01-01T00:00:00Z',
    role: 'admin',
    isVerified: true,
  );

  final user = mapper.fromDto(dto);

  expect(user.id, '1');
  expect(user.createdAt, DateTime.parse('2024-01-01T00:00:00Z'));
  expect(user.role, UserRole.admin); // String → Enum 변환 확인
});

test('UserMapper.fromDto — 알 수 없는 role은 기본값으로', () {
  final mapper = UserMapper();
  final dto = UserResponseDto(
    id: '1', name: 'Kim', email: 'kim@test.com',
    createdAt: '2024-01-01T00:00:00Z',
    role: 'unknown_role', // 서버가 새 role 추가
  );

  final user = mapper.fromDto(dto);

  expect(user.role, UserRole.guest); // 기본값으로 안전하게 처리
});
```

### Result 객체 테스트

```dart
test('CreateUserResult — switch 분기 테스트', () {
  final result = CreateUserSuccess(
    User(id: '1', name: 'Kim', email: 'kim@test.com',
      createdAt: DateTime(2024), role: UserRole.member),
  ) as CreateUserResult;

  final message = switch (result) {
    CreateUserSuccess(:final user) => '생성 완료: ${user.name}',
    CreateUserEmailDuplicate() => '이메일 중복',
    CreateUserFailure(:final message) => '실패: $message',
  };

  expect(message, '생성 완료: Kim');
});
```

### Mock 해야 하는 것

- 없음 — Domain Model, Mapper, Enum은 순수 함수/객체이므로 실제 인스턴스로 테스트

### Mock 하지 말아야 하는 것

- Domain Model — 실제 Freezed 인스턴스 사용 (equality, copyWith 검증)
- Mapper — 실제 변환 로직 검증이 목적

## AI가 자주 하는 실수

### 1. Domain Model에 fromJson/toJson 추가

**AI가 생성하는 코드:**

```dart
// ❌ Domain Model에 JSON 직렬화 추가
@freezed
class Product with _$Product {
  const factory Product({
    required String id,
    required String name,
    @JsonKey(name: 'unit_price') required int unitPrice,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) =>
      _$ProductFromJson(json); // ❌
}
```

**무엇이 잘못되었는가:** Domain Model이 API 스펙(`unit_price`)에 결합되어 API 변경이 비즈니스 로직까지 전파됩니다.

**올바른 접근법:**

```dart
// ✅ Domain Model — 순수
@freezed
class Product with _$Product {
  const factory Product({
    required String id,
    required String name,
    required int unitPrice, // 비즈니스 용어, JSON 무관
  }) = _Product;
}

// ✅ DTO — JSON 전담
@freezed
class ProductResponseDto with _$ProductResponseDto {
  const factory ProductResponseDto({
    required String id,
    required String name,
    @JsonKey(name: 'unit_price') required int unitPrice,
  }) = _ProductResponseDto;

  factory ProductResponseDto.fromJson(Map<String, dynamic> json) =>
      _$ProductResponseDtoFromJson(json);
}
```

### 2. Enum 변환 시 exception throw

**AI가 생성하는 코드:**

```dart
// ❌ 매칭 실패 시 예외 발생
static UserRole fromString(String value) {
  return UserRole.values.firstWhere(
    (e) => e.name == value,
    // orElse 없음 — StateError 발생
  );
}
```

**무엇이 잘못되었는가:** 서버가 새 role 값을 추가하면 앱이 즉시 크래시하며 앱 업데이트 없이 복구 불가합니다.

**올바른 접근법:**

```dart
// ✅ 알 수 없는 값은 기본값으로 안전하게 처리
static UserRole fromString(String value) {
  return UserRole.values.firstWhere(
    (e) => e.name == value,
    orElse: () => UserRole.guest, // 기본값
  );
}
```

### 3. Mapper 없이 DTO에서 직접 도메인 모델 생성

**AI가 생성하는 코드:**

```dart
// ❌ Repository에서 인라인 변환
class UserRepositoryImpl implements UserRepository {
  @override
  Future<User> fetchUser(String id) async {
    final dto = await _api.getUser(id);
    return User(
      id: dto.id,
      name: dto.name,
      email: dto.email,
      createdAt: DateTime.parse(dto.createdAt),
      role: UserRole.fromString(dto.role),
    ); // ❌ 변환 로직이 Repository에 흩어짐
  }
}
```

**무엇이 잘못되었는가:** 동일한 DTO → Model 변환이 여러 Repository 메서드에 중복되어 DTO 변경 시 모든 지점을 수정해야 합니다.

**올바른 접근법:**

```dart
// ✅ Mapper로 변환 로직 집중
@injectable
class UserRepositoryImpl implements UserRepository {
  final UserApi _api;
  final UserMapper _mapper;

  UserRepositoryImpl(this._api, this._mapper);

  @override
  Future<User> fetchUser(String id) async {
    final dto = await _api.getUser(id);
    return _mapper.fromDto(dto); // 변환은 Mapper에 위임
  }
}
```

## 참고 자료

- [freezed 패키지](https://pub.dev/packages/freezed) — 불변 데이터 클래스 및 union type 코드 생성
- [json_serializable 패키지](https://pub.dev/packages/json_serializable) — JSON 직렬화/역직렬화 코드 생성
- [Dart 패턴 매칭](https://dart.dev/language/patterns) — sealed class와 switch 표현식 활용법
