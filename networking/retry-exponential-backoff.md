# Retry & Exponential Backoff

## 개요

네트워크 요청 실패 시 지수 백오프(exponential backoff) 전략으로 재시도하는 패턴입니다. 재시도 간격을 `500ms → 1s → 2s → 4s(cap)`으로 점진적으로 늘려 서버 과부하를 방지하면서 일시적 오류를 자동 복구합니다. BLoC Handler 내부에서 `Result` 패턴과 조합하여 재시도 가능한 에러(네트워크)와 불가능한 에러(4xx)를 구분합니다.

## 사용 시점

- 네트워크 불안정 환경에서 API 호출 안정성을 높여야 할 때
- 서버 일시적 장애(5xx, 타임아웃)에 자동 복구가 필요할 때
- pull-to-refresh, 리스트 로드 등 사용자 인터랙션 기반 요청에서 투명한 재시도가 필요할 때
- WebSocket 연결 끊김 후 재연결 간격을 조절해야 할 때

## 사용하지 말아야 할 때

- 4xx 클라이언트 에러 (400 Bad Request, 401 Unauthorized, 404 Not Found) → 재시도 무의미, 에러 처리 로직으로 분기
- 결제, 송금 등 멱등성이 보장되지 않는 요청 → 멱등성 키(idempotency key)를 먼저 구현한 후 재시도
- 실시간 스트림에서 개별 메시지 유실 → [BLoC Event Transformer](../state-management/bloc-event-transformer.md) throttle/buffer 패턴 사용
- HTTP 클라이언트 레벨에서 이미 재시도가 설정된 경우 (dio interceptor 등) → 중복 재시도 방지

## 필수 구현

### 에러 타입 정의

```dart
// core/util/custom_exception.dart
sealed class CustomException implements Exception {
  final String message;
  const CustomException(this.message);
}

/// 네트워크 연결 실패, 타임아웃 — 재시도 가능
class NetworkErrorException extends CustomException {
  const NetworkErrorException(super.message);
}

/// 서버 에러 (5xx) — 재시도 가능
class ServerErrorException extends CustomException {
  const ServerErrorException(super.message);
}

/// 데이터 없음 — 재시도 불필요
class EmptyDataException extends CustomException {
  const EmptyDataException(super.message);
}

/// 클라이언트 에러 (4xx) — 재시도 불필요
class ClientErrorException extends CustomException {
  const ClientErrorException(super.message);
}
```

### 재시도 가능 여부 판단

```dart
// core/util/retryable.dart
bool isRetryable(CustomException exception) {
  return switch (exception) {
    NetworkErrorException() => true,
    ServerErrorException() => true,
    ClientErrorException() => false,  // 4xx — 재시도 무의미
    EmptyDataException() => false,    // 데이터 없음 — 재시도해도 동일
  };
}
```

### BLoC Handler에서 지수 백오프 구현

```dart
@injectable
class DeliveryListBloc extends Bloc<DeliveryListEvent, DeliveryListState> {
  final GetDeliveryListUseCase _getDeliveryListUseCase;
  static const _maxRetries = 3;
  static const _initialDelay = Duration(milliseconds: 500);
  static const _maxDelay = Duration(milliseconds: 4000);

  DeliveryListBloc(this._getDeliveryListUseCase)
      : super(DeliveryListState.initial()) {
    on<DeliveryListFetchStarted>(
      _onFetchStarted,
      transformer: droppableTransformer(),
    );
    on<DeliveryListRefresh>(
      _onRefresh,
      transformer: droppableTransformer(),
    );
  }

  /// 지수 백오프로 fetch 시도
  Future<Result<List<Delivery>>> _fetchWithRetry({required int page}) async {
    for (var attempt = 0; attempt <= _maxRetries; attempt++) {
      final result = await _getDeliveryListUseCase.call(page: page);

      switch (result) {
        case Ok():
          return result; // 성공 — 즉시 반환
        case Error(:final error):
          // 마지막 시도이거나 재시도 불가능한 에러 — 즉시 실패 반환
          if (attempt == _maxRetries || !isRetryable(error)) {
            return result;
          }
          // 지수 백오프 대기: 500ms → 1s → 2s → 4s(cap)
          final delay = Duration(
            milliseconds: (_initialDelay.inMilliseconds * (1 << attempt))
                .clamp(0, _maxDelay.inMilliseconds),
          );
          await Future.delayed(delay);
      }
    }
    // 도달하지 않지만 컴파일러를 위해
    return Result.error(const NetworkErrorException('Max retries exceeded'));
  }

  Future<void> _onFetchStarted(
    DeliveryListFetchStarted event,
    Emitter<DeliveryListState> emit,
  ) async {
    emit(state.copyWith(
      pagedState: state.pagedState.copyWith(
        fetchStatus: AsyncStatus.loading,
        errorMessage: null,
      ),
    ));

    final result = await _fetchWithRetry(page: 1);

    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchStatus: AsyncStatus.success,
            items: value,
            currentPage: 1,
            hasMore: value.length >= 20,
          ),
        ));
      case Error(:final error):
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchStatus: AsyncStatus.error,
            errorMessage: error.message,
          ),
          lastError: error,
        ));
    }
  }

  /// pull-to-refresh — Completer로 indicator 종료 보장
  Future<void> _onRefresh(
    DeliveryListRefresh event,
    Emitter<DeliveryListState> emit,
  ) async {
    try {
      final result = await _fetchWithRetry(page: 1);

      switch (result) {
        case Ok(:final value):
          emit(state.copyWith(
            pagedState: state.pagedState.copyWith(
              fetchStatus: AsyncStatus.success,
              items: value,
              currentPage: 1,
              hasMore: value.length >= 20,
              errorMessage: null,
            ),
          ));
        case Error(:final error):
          emit(state.copyWith(
            pagedState: state.pagedState.copyWith(
              errorMessage: error.message,
            ),
            lastError: error,
          ));
      }
    } finally {
      event.completer?.complete(); // refresh indicator 항상 종료
    }
  }
}
```

### State에 에러 타입 보존

```dart
class DeliveryListState extends Equatable {
  final PagedState<Delivery> pagedState;
  final CustomException? lastError; // 에러 타입으로 UI 분기 가능

  const DeliveryListState({
    required this.pagedState,
    this.lastError,
  });

  factory DeliveryListState.initial() => DeliveryListState(
    pagedState: PagedState<Delivery>.initial(),
  );

  // UI에서 에러 타입별 분기
  bool get isNetworkError => lastError is NetworkErrorException;
  bool get isServerError => lastError is ServerErrorException;

  // ...copyWith, props 생략
}
```

### Refresh Event에 Completer 포함

```dart
class DeliveryListRefresh extends DeliveryListEvent {
  final Completer<void>? completer;
  const DeliveryListRefresh({this.completer});

  void complete() => completer?.complete();

  @override
  List<Object?> get props => [completer];
}
```

### 핵심 규칙

1. **재시도 가능한 에러만 retry** — `NetworkErrorException`, `ServerErrorException`만 재시도. `ClientErrorException`(4xx)은 즉시 실패
2. **지수 백오프 공식: `initialDelay * 2^attempt`** — 500ms → 1s → 2s → 4s로 간격 증가
3. **maxDelay cap 필수** — 무한정 증가 방지 (보통 4~8초)
4. **maxRetries 제한** — 보통 3회. 무한 재시도 금지
5. **재시도 로직은 BLoC Handler 내부에 위치** — Interceptor와 중복 방지
6. **UI에 재시도 중임을 알리지 않음** — 사용자에게는 단일 loading 상태로 보임. 최종 실패 시에만 에러 표시

## 아키텍처 / 구조

```
lib/
├── core/
│   └── util/
│       ├── custom_exception.dart       # 에러 타입 계층
│       └── retryable.dart              # isRetryable() 판단 함수
│
└── feature/{domain}/presentation/
    └── bloc/{bloc_name}/
        └── {bloc_name}_bloc.dart       # _fetchWithRetry() 메서드
```

## 안티패턴

### 1. 모든 에러에 동일하게 재시도

**잘못된 코드:**

```dart
Future<Result<T>> _fetchWithRetry<T>(Future<Result<T>> Function() action) async {
  for (var i = 0; i <= 3; i++) {
    final result = await action();
    if (result.isOk || i == 3) return result;
    await Future.delayed(Duration(milliseconds: 500 * (1 << i)));
    // ❌ 401 Unauthorized도 재시도 — 토큰 만료는 재시도해도 동일
  }
}
```

**왜 잘못되었는가:** 4xx 에러는 재시도해도 결과가 동일하므로 불필요한 대기만 발생합니다.

**올바른 방법:**

```dart
case Error(:final error):
  if (attempt == _maxRetries || !isRetryable(error)) {
    return result; // 4xx 에러는 즉시 반환
  }
  await Future.delayed(delay);
```

### 2. 고정 간격 재시도

**잘못된 코드:**

```dart
for (var i = 0; i < 3; i++) {
  final result = await action();
  if (result.isOk) return result;
  await Future.delayed(const Duration(seconds: 1)); // ❌ 매번 1초 고정
}
```

**왜 잘못되었는가:** 고정 간격은 "thundering herd" 문제를 유발하여 서버 부하를 악화시킵니다.

**올바른 방법:**

```dart
// 지수 백오프: 500ms → 1s → 2s → 4s
final delay = Duration(
  milliseconds: (_initialDelay.inMilliseconds * (1 << attempt))
      .clamp(0, _maxDelay.inMilliseconds),
);
await Future.delayed(delay);
```

### 3. maxDelay cap 없는 지수 백오프

**잘못된 코드:**

```dart
// ❌ cap 없음 — retry 10회 시 500ms * 2^9 = 256초(4분) 대기
final delay = Duration(milliseconds: 500 * (1 << attempt));
```

**왜 잘못되었는가:** cap 없이 retry 10회 시 256초(4분) 대기가 발생합니다.

**올바른 방법:**

```dart
final delay = Duration(
  milliseconds: (500 * (1 << attempt)).clamp(0, 4000), // 최대 4초
);
```

### 4. Interceptor와 BLoC에서 이중 재시도

**잘못된 코드:**

```dart
// Dio Interceptor — 3회 재시도
class RetryInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // 3회 재시도 로직
  }
}

// BLoC Handler — 또 3회 재시도
Future<void> _onFetch(...) async {
  final result = await _fetchWithRetry(page: 1); // ❌ 총 9회 요청 가능
}
```

**왜 잘못되었는가:** 양쪽에서 3회씩 재시도하면 최악의 경우 `3 x 3 = 9회` 요청이 발생합니다.

**올바른 방법:**

재시도 레이어는 **한 곳에서만** 처리합니다. BLoC에서 재시도하면 Interceptor에서는 재시도하지 않고, 반대도 마찬가지입니다.

```dart
// Interceptor — 재시도 없이 에러 변환만 담당
class ErrorMappingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // 에러 타입 변환 후 전파 — 재시도 안 함
    handler.reject(err);
  }
}

// BLoC — 유일한 재시도 레이어
final result = await _fetchWithRetry(page: 1);
```

## 의사결정 기준

### 재시도 파라미터 선택

| 파라미터 | 일반 API | 결제/중요 요청 | 백그라운드 동기화 |
|---|---|---|---|
| maxRetries | 3 | 0 (재시도 안 함) | 5 |
| initialDelay | 500ms | — | 1000ms |
| maxDelay | 4s | — | 30s |
| 재시도 대상 | Network, Server(5xx) | 없음 | Network, Server(5xx) |

### 재시도 레이어 선택

| 기준 | BLoC Handler | Dio Interceptor |
|---|---|---|
| 에러 타입별 분기 | ✅ `CustomException` 타입 활용 | ⚠️ HTTP 상태코드만 |
| 상태 업데이트 | ✅ emit으로 로딩 상태 관리 | ❌ 불가 |
| 테스트 용이성 | ✅ blocTest로 시나리오 검증 | ⚠️ 별도 설정 필요 |
| 재사용성 | 각 BLoC에서 개별 적용 | 전역 적용 |

**의사결정 규칙:** 에러 타입별 세밀한 제어가 필요하면 BLoC Handler. 모든 API에 동일한 재시도 정책을 적용하려면 Interceptor. **절대 양쪽에서 동시에 재시도하지 않음.**

## 테스트 전략

```dart
void main() {
  late DeliveryListBloc bloc;
  late FakeDeliveryRepository fakeRepository;

  setUp(() {
    fakeRepository = FakeDeliveryRepository();
    bloc = DeliveryListBloc(GetDeliveryListUseCase(fakeRepository));
  });

  tearDown(() => bloc.close());

  blocTest<DeliveryListBloc, DeliveryListState>(
    '네트워크 에러 후 재시도 성공 시 데이터가 로드된다',
    build: () {
      var callCount = 0;
      fakeRepository.onFetch = (page) {
        callCount++;
        if (callCount <= 2) {
          return Result.error(const NetworkErrorException('타임아웃'));
        }
        return Result.ok([Delivery(id: '1')]);
      };
      return bloc;
    },
    act: (bloc) => bloc.add(const DeliveryListFetchStarted()),
    wait: const Duration(seconds: 3), // 백오프 대기 시간 포함
    expect: () => [
      predicate<DeliveryListState>(
        (s) => s.pagedState.fetchStatus == AsyncStatus.loading,
      ),
      predicate<DeliveryListState>(
        (s) =>
            s.pagedState.fetchStatus == AsyncStatus.success &&
            s.pagedState.items.length == 1,
      ),
    ],
  );

  blocTest<DeliveryListBloc, DeliveryListState>(
    'ClientError(4xx)는 재시도 없이 즉시 실패한다',
    build: () {
      fakeRepository.errorToThrow = const ClientErrorException('404 Not Found');
      return bloc;
    },
    act: (bloc) => bloc.add(const DeliveryListFetchStarted()),
    expect: () => [
      predicate<DeliveryListState>(
        (s) => s.pagedState.fetchStatus == AsyncStatus.loading,
      ),
      predicate<DeliveryListState>(
        (s) =>
            s.pagedState.fetchStatus == AsyncStatus.error &&
            !s.isNetworkError, // 네트워크 에러가 아님
      ),
    ],
  );

  blocTest<DeliveryListBloc, DeliveryListState>(
    'maxRetries 초과 시 최종 에러를 반환한다',
    build: () {
      fakeRepository.errorToThrow = const NetworkErrorException('타임아웃');
      return bloc;
    },
    act: (bloc) => bloc.add(const DeliveryListFetchStarted()),
    wait: const Duration(seconds: 10), // 500ms + 1s + 2s + 4s 백오프 대기
    expect: () => [
      predicate<DeliveryListState>(
        (s) => s.pagedState.fetchStatus == AsyncStatus.loading,
      ),
      predicate<DeliveryListState>(
        (s) =>
            s.pagedState.fetchStatus == AsyncStatus.error &&
            s.isNetworkError,
      ),
    ],
  );
}
```

### Mock 해야 하는 것

- UseCase / Repository (호출 횟수와 에러 타입 제어)

### Mock 하지 말아야 하는 것

- **지수 백오프 로직** — 실제 delay 포함 테스트 (`wait` 파라미터로 충분한 시간 설정)
- **isRetryable 판단** — 실제 에러 타입으로 검증

## AI가 자주 하는 실수

### 1. 에러 타입 구분 없이 모든 에러 재시도

**AI가 생성하는 코드:**

```dart
Future<Result<T>> fetchWithRetry<T>(Future<Result<T>> Function() fn) async {
  for (var i = 0; i < 3; i++) {
    final result = await fn();
    if (result.isOk) return result;
    await Future.delayed(Duration(seconds: i + 1)); // ❌ 모든 에러에 재시도
  }
  return fn();
}
```

**무엇이 잘못되었는가:** 4xx 에러는 재시도해도 결과가 동일하여 불필요한 대기만 발생합니다.

**올바른 접근법:**

```dart
case Error(:final error):
  if (attempt == _maxRetries || !isRetryable(error)) {
    return result; // 재시도 불가능한 에러는 즉시 반환
  }
  await Future.delayed(delay);
```

### 2. 재시도 중 상태를 매번 emit

**AI가 생성하는 코드:**

```dart
for (var attempt = 0; attempt <= maxRetries; attempt++) {
  emit(state.copyWith(
    retryCount: attempt,
    status: AsyncStatus.loading, // ❌ 매 시도마다 emit → UI 깜빡임
  ));
  final result = await _useCase.call();
  // ...
}
```

**무엇이 잘못되었는가:** 매 재시도마다 emit하면 UI가 `loading → error → loading → ...`로 깜빡입니다.

**올바른 접근법:**

```dart
// loading은 최초 1회만 emit
emit(state.copyWith(fetchStatus: AsyncStatus.loading));

// 재시도 로직은 내부에서 조용히 처리
final result = await _fetchWithRetry(page: 1);

// 최종 결과만 emit
switch (result) {
  case Ok(:final value):
    emit(state.copyWith(fetchStatus: AsyncStatus.success, items: value));
  case Error(:final error):
    emit(state.copyWith(fetchStatus: AsyncStatus.error, errorMessage: error.message));
}
```

## 참고 자료

- [Exponential Backoff — Google Cloud](https://cloud.google.com/memorystore/docs/redis/exponential-backoff) — 지수 백오프 공식 설명
- [flutter_bloc 패키지](https://pub.dev/packages/flutter_bloc) — BLoC Handler에서 재시도 구현
- [dio 패키지](https://pub.dev/packages/dio) — HTTP 클라이언트 Interceptor
