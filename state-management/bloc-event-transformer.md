# BLoC Event Transformer

## 개요

`bloc_concurrency`와 `stream_transform` 패키지를 조합하여 BLoC 이벤트의 동시성을 선언적으로 제어하는 패턴입니다. 검색 debounce, 버튼 연타 방지, 실시간 스트림 throttle 등 실무에서 반복되는 동시성 문제를 Transformer 한 줄로 해결합니다. 모든 `on<Event>` 등록 시 Transformer를 명시적으로 지정하는 것이 핵심입니다.

## 사용 시점

- 검색/자동완성 입력에서 타이핑 중 불필요한 API 호출을 방지해야 할 때
- 버튼 연타, 결제 중복 실행 등을 차단해야 할 때
- 실시간 WebSocket/스트림 데이터를 일정 간격으로 배치 처리해야 할 때
- 리스트 fetch, pull-to-refresh 중 재요청을 무시해야 할 때
- 순서가 보장되어야 하는 연속 요청(큐잉)이 필요할 때

## 사용하지 말아야 할 때

- 이벤트 변환이 필요 없는 단순 상태 토글 → [Cubit](https://bloclibrary.dev/bloc-concepts/#cubit) 사용
- 위젯 로컬 상태 (애니메이션, 폼 입력) → [Flutter Hooks](local-state-management-by-flutter-hook.md) 사용
- BLoC 없이 Stream 자체를 변환해야 할 때 → `stream_transform` 패키지 직접 사용

## 필수 구현

### 기본 Transformer 유틸리티 파일

프로젝트 전역에서 재사용할 Transformer를 하나의 파일에 정의합니다.

```dart
// core/util/bloc_event_transformer.dart
import 'package:bloc_concurrency/bloc_concurrency.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:stream_transform/stream_transform.dart';

/// 모든 이벤트를 동시에 처리 (기본 동작과 동일 — 명시적으로 쓸 때만 사용)
EventTransformer<E> concurrentTransformer<E>() => concurrent<E>();

/// 이벤트를 순서대로 하나씩 처리 (큐 방식)
EventTransformer<E> sequentialTransformer<E>() => sequential<E>();

/// 새 이벤트가 들어오면 이전 처리를 취소하고 재시작 (switchMap)
EventTransformer<E> restartableTransformer<E>() => restartable<E>();

/// 처리 중일 때 들어오는 이벤트를 무시 (중복 클릭 방지)
EventTransformer<E> droppableTransformer<E>() => droppable<E>();

/// 일정 시간 입력이 멈춘 후에만 처리 (검색 입력 등)
EventTransformer<E> debounceTransformer<E>({
  Duration duration = const Duration(milliseconds: 300),
}) {
  return (events, mapper) => events.debounce(duration).switchMap(mapper);
}

/// 일정 간격으로만 이벤트를 처리 (스크롤, 리사이즈 등)
EventTransformer<E> throttleTransformer<E>({
  Duration duration = const Duration(milliseconds: 300),
}) {
  return (events, mapper) => events.throttle(duration).switchMap(mapper);
}

/// debounce + droppable: 입력 대기 후 중복 요청 무시
EventTransformer<E> debounceDroppableTransformer<E>({
  Duration duration = const Duration(milliseconds: 300),
}) {
  return (events, mapper) {
    return droppable<E>().call(events.debounce(duration), mapper);
  };
}

/// debounce + restartable: 입력 대기 후 이전 요청 취소 (검색 API에 최적)
EventTransformer<E> debounceRestartableTransformer<E>({
  Duration duration = const Duration(milliseconds: 300),
}) {
  return (events, mapper) {
    return restartable<E>().call(events.debounce(duration), mapper);
  };
}
```

### 검색 BLoC — debounceRestartable 적용

```dart
@injectable
class FoodSearchBloc extends Bloc<FoodSearchEvent, FoodSearchState> {
  final SearchFoodUseCase _searchFoodUseCase;

  FoodSearchBloc(this._searchFoodUseCase)
      : super(const FoodSearchState()) {
    // debounce(300ms) → 이전 요청 취소 → 최신 쿼리만 API 호출
    on<FoodSearchQueryChanged>(
      _onQueryChanged,
      transformer: debounceRestartableTransformer(),
    );
    on<FoodSearchCleared>(
      _onCleared,
      transformer: droppableTransformer(),
    );
  }

  Future<void> _onQueryChanged(
    FoodSearchQueryChanged event,
    Emitter<FoodSearchState> emit,
  ) async {
    final query = event.query.trim();
    if (query.isEmpty) {
      emit(const FoodSearchState()); // initial로 리셋
      return;
    }

    emit(state.copyWith(status: FoodSearchStatus.loading, query: query));

    final result = await _searchFoodUseCase.call(query: query);
    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(status: FoodSearchStatus.success, results: value));
      case Error(:final error):
        emit(state.copyWith(
          status: FoodSearchStatus.failure,
          errorMessage: error.message,
        ));
    }
  }
}
```

### 결제 BLoC — droppable 적용

```dart
@injectable
class PaymentBloc extends Bloc<PaymentEvent, PaymentState> {
  final SubmitPaymentUseCase _submitPaymentUseCase;

  PaymentBloc(this._submitPaymentUseCase) : super(const PaymentState()) {
    // 결제 처리 중 모든 후속 요청을 무시
    on<PaymentRequested>(
      _onPaymentRequested,
      transformer: droppableTransformer(),
    );
  }

  Future<void> _onPaymentRequested(
    PaymentRequested event,
    Emitter<PaymentState> emit,
  ) async {
    emit(state.copyWith(isProcessing: true));

    final result = await _submitPaymentUseCase.call(
      amount: event.amount,
      methodId: event.methodId,
    );

    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(isProcessing: false, paymentResult: value));
      case Error(:final error):
        emit(state.copyWith(isProcessing: false, errorMessage: error.message));
    }
  }
}
```

### 실시간 대시보드 — throttle 적용

```dart
@injectable
class OrderDashboardBloc extends Bloc<DashboardEvent, DashboardState> {
  final WatchOrderStreamUseCase _watchOrderStreamUseCase;
  StreamSubscription<Order>? _orderSubscription;

  OrderDashboardBloc(this._watchOrderStreamUseCase)
      : super(const DashboardState()) {
    on<DashboardStarted>(_onStarted, transformer: droppableTransformer());
    // 500ms 간격으로 배치 처리 — WebSocket 이벤트 폭주 방지
    on<OrderBatchReady>(
      _onOrderBatchReady,
      transformer: throttleTransformer(
        duration: const Duration(milliseconds: 500),
      ),
    );
    on<DashboardStopped>(_onStopped, transformer: droppableTransformer());
  }

  Future<void> _onStarted(
    DashboardStarted event,
    Emitter<DashboardState> emit,
  ) async {
    emit(state.copyWith(status: DashboardStatus.loading));
    _orderSubscription = _watchOrderStreamUseCase.call().listen(
      (order) => add(const OrderBatchReady()),
    );
    emit(state.copyWith(status: DashboardStatus.connected));
  }

  @override
  Future<void> close() {
    _orderSubscription?.cancel();
    return super.close();
  }
}
```

### 핵심 규칙

1. **모든 `on<Event>`에 Transformer를 명시적으로 지정** — 기본값(`concurrent`)에 의존하지 않음
2. **Transformer 유틸리티는 단일 파일에 정의** — `core/util/bloc_event_transformer.dart`
3. **조합 Transformer는 inner(outer) 순서** — `droppable().call(events.debounce(duration), mapper)` = debounce 후 droppable 적용
4. **Duration은 시나리오에 맞게 조정** — 검색 300ms, 버튼 500ms, 결제 800ms, 스트림 배치 500ms
5. **Transformer 테스트는 실제 동작 포함** — mock하지 않고 실제 Transformer로 테스트

## 아키텍처 / 구조

```
lib/
├── core/
│   └── util/
│       └── bloc_event_transformer.dart   # 모든 Transformer 정의
│
└── feature/{domain}/presentation/
    └── bloc/{bloc_name}/
        └── {bloc_name}_bloc.dart         # on<Event>에서 Transformer 참조
```

## 안티패턴

### 1. Transformer 미지정 (기본 concurrent 의존)

**잘못된 코드:**

```dart
on<SearchQueryChanged>(_onSearchQueryChanged); // Transformer 없음
```

**왜 잘못되었는가:** 기본 `concurrent`로 인해 모든 입력이 병렬 실행되어 응답 순서에 따라 잘못된 결과가 표시됩니다.

**올바른 방법:**

```dart
on<SearchQueryChanged>(
  _onSearchQueryChanged,
  transformer: debounceRestartableTransformer(), // 마지막 입력만 처리, 이전 취소
);
```

### 2. Handler 내부에서 수동 debounce 구현

**잘못된 코드:**

```dart
Timer? _debounceTimer;

Future<void> _onSearchQueryChanged(
  SearchQueryChanged event,
  Emitter<SearchState> emit,
) async {
  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 300), () async {
    // ❌ Timer 콜백에서 emit 호출 — emit은 handler scope 밖에서 호출 불가
    emit(state.copyWith(status: AsyncStatus.loading));
    final result = await _searchUseCase.call(event.query);
    // ...
  });
}
```

**왜 잘못되었는가:** Timer 콜백은 핸들러 종료 후 실행되므로 `emit` 호출 시 `StateError`가 발생합니다.

**올바른 방법:**

```dart
on<SearchQueryChanged>(
  _onSearchQueryChanged,
  transformer: debounceRestartableTransformer(), // Transformer가 debounce 담당
);

// Handler는 순수 비즈니스 로직만
Future<void> _onSearchQueryChanged(
  SearchQueryChanged event,
  Emitter<SearchState> emit,
) async {
  emit(state.copyWith(status: AsyncStatus.loading));
  final result = await _searchUseCase.call(event.query);
  // ...
}
```

### 3. 조합 Transformer에서 순서 혼동

**잘못된 코드:**

```dart
// ❌ debounce를 안쪽에 넣음 — droppable이 먼저 이벤트를 버린 후 debounce 적용
EventTransformer<E> wrongOrder<E>(Duration duration) {
  return (events, mapper) {
    return events.debounce(duration).switchMap(mapper); // restartable 동작이지 droppable이 아님
  };
}
```

**왜 잘못되었는가:** 조합 Transformer는 **outer(inner)** 순서로, debounce 필터링 후 droppable/restartable을 적용해야 합니다.

**올바른 방법:**

```dart
// ✅ debounce → droppable: 입력 대기 후 중복 방지
EventTransformer<E> debounceDroppableTransformer<E>({
  Duration duration = const Duration(milliseconds: 300),
}) {
  return (events, mapper) {
    return droppable<E>().call(events.debounce(duration), mapper);
    //     ^^^^^^^^^^^outer    ^^^^^^^^^^^^^^^^^^^^^^inner
  };
}
```

### 4. 모든 이벤트에 같은 Transformer 일괄 적용

**잘못된 코드:**

```dart
// ❌ 글로벌 Transformer 설정으로 모든 이벤트를 droppable 처리
Bloc.transformer = droppable(); // 전역 설정

class MyBloc extends Bloc<MyEvent, MyState> {
  MyBloc() : super(const MyState()) {
    on<DataFetched>(_onDataFetched);  // droppable — OK
    on<SearchChanged>(_onSearch);     // droppable — 검색에는 restartable이 필요
  }
}
```

**왜 잘못되었는가:** 이벤트마다 최적의 동시성 전략이 다르므로 개별 지정이 필요합니다.

**올바른 방법:**

```dart
class MyBloc extends Bloc<MyEvent, MyState> {
  MyBloc() : super(const MyState()) {
    on<DataFetched>(_onDataFetched, transformer: droppableTransformer());
    on<SearchChanged>(_onSearch, transformer: debounceRestartableTransformer());
    on<ItemDeleted>(_onDeleted, transformer: droppableTransformer());
  }
}
```

## 의사결정 기준

### Transformer 선택 매트릭스

| 시나리오 | Transformer | 동작 | 예시 |
|---|---|---|---|
| 검색/자동완성 입력 | `debounceRestartableTransformer(300ms)` | 입력 멈춘 후 최신 쿼리만 처리, 이전 API 호출 취소 | 음식 검색, 주소 자동완성 |
| 버튼 연타 방지 | `droppableTransformer()` | 처리 중 후속 이벤트 무시 | 좋아요, 삭제, 결제 |
| 데이터 로드 (중복 방지) | `droppableTransformer()` | 진행 중이면 새 요청 무시 | 리스트 fetch, 초기 로딩 |
| 실시간 스트림 배치 | `throttleTransformer(500ms)` | 일정 간격으로만 이벤트 통과 | WebSocket 주문, 알림 |
| 스크롤 이벤트 | `throttleTransformer(300ms)` | 빠른 스크롤 중 이벤트 제한 | 무한 스크롤, 위치 추적 |
| 순서 보장 필요 | `sequentialTransformer()` | 이벤트를 큐잉, 순차 처리 | 채팅 메시지 전송, 순차 업로드 |
| 최신 요청만 처리 | `restartableTransformer()` | 새 이벤트 오면 이전 취소 | 필터 변경, 탭 전환 |
| 입력 대기 + 중복 방지 | `debounceDroppableTransformer(300ms)` | debounce 후 처리 중 무시 | 폼 유효성 검사 API |

### 의사결정 플로차트

```
이벤트가 사용자 입력(타이핑)인가?
├── YES → debounce 필요
│   ├── 이전 요청을 취소해야 하는가? → debounceRestartableTransformer
│   └── 처리 중 무시해도 되는가? → debounceDroppableTransformer
└── NO → debounce 불필요
    ├── 중복 실행을 방지해야 하는가? → droppableTransformer
    ├── 일정 간격으로 제한해야 하는가? → throttleTransformer
    ├── 순서가 보장되어야 하는가? → sequentialTransformer
    └── 최신 것만 필요한가? → restartableTransformer
```

### restartable vs droppable

| 기준 | `restartable` | `droppable` |
|---|---|---|
| 이전 처리 | **취소** (switchMap) | 완료까지 대기 |
| 새 이벤트 | **즉시 시작** | **무시** |
| 적합한 상황 | 최신 결과가 중요 (검색, 필터) | 첫 요청 완료가 중요 (삭제, 결제) |
| API 호출 | 이전 호출 취소 → 불필요한 응답 처리 없음 | 진행 중 호출은 완료 → 결과 보장 |

**의사결정 규칙:** "최신 값이 정답"이면 `restartable`, "첫 요청이 완료되어야"이면 `droppable`.

## 테스트 전략

Event Transformer는 mock하지 않고 **실제 Transformer를 포함하여** 테스트합니다.

```dart
void main() {
  late FoodSearchBloc bloc;
  late FakeSearchRepository fakeRepository;

  setUp(() {
    fakeRepository = FakeSearchRepository();
    bloc = FoodSearchBloc(SearchFoodUseCase(fakeRepository));
  });

  tearDown(() => bloc.close());

  blocTest<FoodSearchBloc, FoodSearchState>(
    'debounceRestartable — 빠른 연속 입력 시 마지막 쿼리만 처리',
    build: () {
      fakeRepository.resultToReturn = [FoodItem(name: 'Pizza')];
      return bloc;
    },
    act: (bloc) async {
      // 빠르게 연속 입력 — debounce로 앞 2개는 무시됨
      bloc.add(const FoodSearchQueryChanged(query: 'P'));
      bloc.add(const FoodSearchQueryChanged(query: 'Pi'));
      bloc.add(const FoodSearchQueryChanged(query: 'Pizza'));
      // debounce 시간(300ms) + 여유 대기
      await Future.delayed(const Duration(milliseconds: 500));
    },
    expect: () => [
      // 'P', 'Pi'는 debounce로 무시 — 'Pizza'만 처리
      const FoodSearchState(status: FoodSearchStatus.loading, query: 'Pizza'),
      FoodSearchState(
        status: FoodSearchStatus.success,
        query: 'Pizza',
        results: [FoodItem(name: 'Pizza')],
      ),
    ],
    wait: const Duration(milliseconds: 600),
  );

  blocTest<FoodSearchBloc, FoodSearchState>(
    'droppable — 처리 중 이벤트는 무시된다',
    build: () {
      fakeRepository.delay = const Duration(milliseconds: 200);
      fakeRepository.resultToReturn = [FoodItem(name: 'Result')];
      return bloc;
    },
    act: (bloc) async {
      bloc.add(const FoodSearchCleared()); // 처리 시작
      await Future.delayed(const Duration(milliseconds: 50));
      bloc.add(const FoodSearchCleared()); // 무시됨
    },
    expect: () => [
      const FoodSearchState(), // cleared 결과 1회만
    ],
    wait: const Duration(milliseconds: 300),
  );
}
```

### Mock 규칙

- **Mock O:** UseCase (BLoC 단위 테스트에서 Repository 격리 시)
- **Mock X:** Event Transformer (실제 동작으로 검증), Duration (`wait` 파라미터로 대기 시간 설정)

## AI가 자주 하는 실수

> **참고:** 안티패턴 섹션의 패턴 1~4도 AI가 자주 생성하는 코드입니다. 반드시 함께 확인하세요.

### 1. throttle과 debounce를 혼동

**AI가 생성하는 코드:**

```dart
// ❌ 검색에 throttle 적용
on<SearchQueryChanged>(
  _onSearch,
  transformer: throttleTransformer(duration: const Duration(milliseconds: 300)),
);
```

**무엇이 잘못되었는가:** `throttle`은 첫 이벤트를 즉시 처리하고 이후 300ms 동안 무시합니다. 사용자가 "P" 입력 시 바로 "P"로 검색되고, 0.3초 후 "Pizza"가 처리됩니다. 검색에는 **입력이 멈춘 후** 처리하는 `debounce`가 올바릅니다.

**올바른 접근법:**

```dart
// 검색 — debounce (입력 멈춘 후 처리)
on<SearchQueryChanged>(
  _onSearch,
  transformer: debounceRestartableTransformer(),
);

// 스크롤/리사이즈 — throttle (일정 간격으로 처리)
on<ScrollPositionChanged>(
  _onScroll,
  transformer: throttleTransformer(),
);
```

## 참고 자료

- [bloc_concurrency 패키지](https://pub.dev/packages/bloc_concurrency) — droppable, sequential, restartable, concurrent Transformer
- [stream_transform 패키지](https://pub.dev/packages/stream_transform) — debounce, throttle 등 스트림 변환 유틸리티
- [flutter_bloc 패키지](https://pub.dev/packages/flutter_bloc) — EventTransformer 타입 정의
- [BLoC 공식 문서 — Event Transformer](https://bloclibrary.dev/bloc-concepts/#event-transformers) — 공식 가이드
