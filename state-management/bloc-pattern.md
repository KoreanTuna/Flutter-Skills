# BLoC 패턴

## 개요

`flutter_bloc` 패키지를 사용하여 비즈니스 로직을 UI에서 완전히 분리하는 상태 관리 패턴입니다. Event → BLoC → State 단방향 흐름을 따르며, Event Transformer를 통해 동시성 제어(debounce, throttle, sequential)를 선언적으로 처리합니다. 테스트가 용이하고 복잡한 비동기 플로우를 안전하게 관리할 수 있습니다.

## 사용 시점

- 비동기 API 호출, 페이지네이션, 폼 제출 등 비즈니스 로직이 포함된 상태 관리가 필요할 때
- 이벤트 단위로 동시성 제어(debounce, throttle, droppable)가 필요할 때
- 여러 화면에서 동일한 비즈니스 상태를 공유해야 할 때
- 이벤트 소싱 기반으로 상태 변화를 추적하고 테스트해야 할 때

## 사용하지 말아야 할 때

- 탭 인덱스, 토글, 필터 등 이벤트 변환이 불필요한 단순 상태 → [Cubit](https://bloclibrary.dev/bloc-concepts/#cubit) 사용
- 애니메이션 컨트롤러, TextEditingController 등 Widget 로컬 상태 → [Flutter Hooks](local-state-management-by-flutter-hook.md) 사용
- 앱 전역 단순 상태(테마, 로케일) → Cubit + `@lazySingleton` 사용

## 필수 구현

### BLoC 파일 구조 — part/part of 분리

```dart
// card_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:equatable/equatable.dart';
import 'package:injectable/injectable.dart';

part 'card_event.dart';
part 'card_state.dart';

@injectable
class CardBloc extends Bloc<CardEvent, CardState> {
  final FetchCardsUseCase _fetchCardsUseCase;
  final DeleteCardUseCase _deleteCardUseCase;

  CardBloc(this._fetchCardsUseCase, this._deleteCardUseCase)
      : super(const CardState()) {
    on<CardListFetched>(_onCardListFetched,
        transformer: droppable()); // 중복 요청 방지
    on<CardDeleted>(_onCardDeleted,
        transformer: throttleDroppable(const Duration(milliseconds: 500)));
  }
}
```

```dart
// card_event.dart
part of 'card_bloc.dart';

sealed class CardEvent extends Equatable {
  const CardEvent();
  @override
  List<Object?> get props => [];
}

// 과거분사/명사로 네이밍 — "사용자가 한 행위"를 표현
class CardListFetched extends CardEvent {
  const CardListFetched();
}

class CardDeleted extends CardEvent {
  final String cardId;
  const CardDeleted({required this.cardId});
  @override
  List<Object?> get props => [cardId];
}
```

```dart
// card_state.dart
part of 'card_bloc.dart';

class CardState extends Equatable {
  final AsyncStatus fetchStatus;
  final AsyncStatus deleteStatus;
  final List<Card> cards;
  final String? errorMessage;

  const CardState({
    this.fetchStatus = AsyncStatus.initial,
    this.deleteStatus = AsyncStatus.initial,
    this.cards = const [],
    this.errorMessage,
  });

  // nullable 필드 초기화를 위한 clearX 플래그
  CardState copyWith({
    AsyncStatus? fetchStatus,
    AsyncStatus? deleteStatus,
    List<Card>? cards,
    String? errorMessage,
    bool clearErrorMessage = false,
  }) {
    return CardState(
      fetchStatus: fetchStatus ?? this.fetchStatus,
      deleteStatus: deleteStatus ?? this.deleteStatus,
      cards: cards ?? this.cards,
      errorMessage: clearErrorMessage ? null : errorMessage ?? this.errorMessage,
    );
  }

  @override
  List<Object?> get props => [fetchStatus, deleteStatus, cards, errorMessage];
}
```

### Event Transformer — 동시성 제어

**기본값(concurrent)에 의존하지 말고 모든 이벤트 핸들러에 명시적으로 Transformer를 지정하세요.** 시나리오별 선택 가이드: [BLoC Event Transformer](bloc-event-transformer.md)

### Handler 패턴

```dart
Future<void> _onCardListFetched(
  CardListFetched event,
  Emitter<CardState> emit,
) async {
  // 1. Guard clause — 중복 요청 차단
  if (state.fetchStatus == AsyncStatus.loading) return;

  // 2. Loading emit (에러 메시지 초기화)
  emit(state.copyWith(
    fetchStatus: AsyncStatus.loading,
    clearErrorMessage: true,
  ));

  // 3. UseCase 호출
  final result = await _fetchCardsUseCase.call();

  // 4. Result 패턴 매칭
  switch (result) {
    case Ok(:final value):
      emit(state.copyWith(
        fetchStatus: AsyncStatus.success,
        cards: value,
      ));
    case Error(:final error):
      emit(state.copyWith(
        fetchStatus: AsyncStatus.error,
        errorMessage: error.message,
      ));
  }
}
```

### Completer를 활용한 비동기 조정

여러 BLoC의 refresh를 동시에 기다려야 할 때 `Completer`를 사용합니다.

```dart
// Event에 Completer 파라미터 추가
class CardListFetched extends CardEvent {
  final Completer<void>? completer;
  const CardListFetched({this.completer});
  @override
  List<Object?> get props => [completer];
}

// Handler — finally에서 Completer 완료
Future<void> _onCardListFetched(
  CardListFetched event,
  Emitter<CardState> emit,
) async {
  try {
    emit(state.copyWith(fetchStatus: AsyncStatus.loading));
    final result = await _fetchCardsUseCase.call();
    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(fetchStatus: AsyncStatus.success, cards: value));
      case Error(:final error):
        emit(state.copyWith(fetchStatus: AsyncStatus.error, errorMessage: error.message));
    }
  } finally {
    event.completer?.complete(); // 항상 완료 보장
  }
}

// View에서 여러 BLoC 동시 refresh
Future<void> _onRefresh(BuildContext context) async {
  final cardCompleter = Completer<void>();
  final orderCompleter = Completer<void>();

  context.read<CardBloc>().add(CardListFetched(completer: cardCompleter));
  context.read<OrderBloc>().add(OrderListFetched(completer: orderCompleter));

  await Future.wait([cardCompleter.future, orderCompleter.future]);
}
```

### Stream Subscription 관리

외부 스트림을 구독하는 BLoC은 `close()`에서 반드시 구독을 취소합니다.

```dart
@injectable
class NotificationBloc extends Bloc<NotificationEvent, NotificationState> {
  final NotificationCoordinator _coordinator;
  late final StreamSubscription<NotificationUpdate> _subscription;

  NotificationBloc(this._coordinator)
      : super(const NotificationState()) {
    _subscription = _coordinator.updates.listen(
      (update) => add(_NotificationReceived(update: update)),
    );
    on<_NotificationReceived>(_onNotificationReceived);
  }

  @override
  Future<void> close() {
    _subscription.cancel(); // 필수 — 메모리 누수 방지
    return super.close();
  }
}
```

### 핵심 규칙

1. **Event는 `sealed class`로 정의** — 과거분사/명사 네이밍 (`DataFetched`, `ItemDeleted`, `ResultConsumed`)
2. **State는 `Equatable` 확장 + `const` 생성자 + 기본값** — State는 순수 데이터 컨테이너 (단순 computed getter만 허용)
3. **모든 BLoC은 `@injectable`로 DI 등록** — GetIt으로 생명주기 관리
4. **nullable 필드 초기화에 `clearX` 플래그 사용** — `copyWith`에서 명시적 null 전달 불가 문제 해결
5. **Handler는 Guard → Loading → UseCase → Result 패턴 준수** — finally에서 Completer 완료
6. **Event Transformer는 시나리오에 맞게 명시적 지정** — 기본값(concurrent)에 의존하지 않음

## 아키텍처 / 구조

```
lib/feature/{domain}/presentation/
├── bloc/{bloc_name}/
│   ├── {bloc_name}_bloc.dart        # BLoC 클래스 + on<Event> 등록
│   ├── {bloc_name}_event.dart       # part of — sealed Event
│   └── {bloc_name}_state.dart       # part of — State + copyWith
├── view/{view_name}/
│   ├── {view_name}_view.dart        # BLoC 소비 (context.select, BlocBuilder)
│   └── widget/
│       └── {widget_name}.dart       # 순수 UI (BLoC 접근 금지)
└── {feature}_screen.dart            # 라우트 진입점, useEffect에서 초기 이벤트 발행
```

**Sub-feature가 있는 경우:**

```
lib/feature/{domain}/presentation/
├── {sub_feature_a}/
│   ├── bloc/{bloc_name}/            # sub-feature 전용 BLoC
│   ├── view/
│   └── {sub_feature_a}_screen.dart
├── bloc/{bloc_name}/                # 여러 sub-feature 공유 BLoC
├── view/{view_name}/
└── {feature}_screen.dart
```

## 안티패턴

### 1. BLoC에서 BuildContext 접근

**잘못된 코드:**

```dart
class CardBloc extends Bloc<CardEvent, CardState> {
  Future<void> _onCardDeleted(CardDeleted event, Emitter<CardState> emit) async {
    // ❌ BLoC에서 BuildContext 사용
    final navigator = Navigator.of(event.context);
    await _deleteCardUseCase.call(event.cardId);
    navigator.pop();
  }
}
```

**왜 잘못되었는가:** BLoC이 UI 프레임워크에 의존하면 테스트 불가능하고 관심사 분리가 깨집니다.

**올바른 방법:**

```dart
// BLoC — 상태만 변경
Future<void> _onCardDeleted(CardDeleted event, Emitter<CardState> emit) async {
  final result = await _deleteCardUseCase.call(event.cardId);
  switch (result) {
    case Ok():
      emit(state.copyWith(deleteStatus: AsyncStatus.success));
    case Error(:final error):
      emit(state.copyWith(errorMessage: error.message));
  }
}

// View — BlocListener에서 네비게이션 처리
BlocListener<CardBloc, CardState>(
  listenWhen: (prev, curr) => prev.deleteStatus != curr.deleteStatus,
  listener: (context, state) {
    if (state.deleteStatus == AsyncStatus.success) {
      Navigator.of(context).pop();
    }
  },
  child: // ...
)
```

### 2. Route에서 초기 이벤트 발행

**잘못된 코드:**

```dart
// ❌ Route 정의에서 초기 이벤트 발행
GoRoute(
  path: '/cards',
  builder: (context, state) => BlocProvider(
    create: (_) => locator<CardBloc>()..add(const CardListFetched()), // 금지
    child: const CardScreen(),
  ),
)
```

**왜 잘못되었는가:** Route에서 이벤트 발행 시 Widget 생명주기와 분리되어 race condition이 발생할 수 있습니다.

**올바른 방법:**

```dart
// Route — BLoC 제공만
GoRoute(
  path: '/cards',
  builder: (context, state) => BlocProvider(
    create: (_) => locator<CardBloc>(),
    child: const CardScreen(),
  ),
)

// Screen — useEffect에서 초기 이벤트 발행
class CardScreen extends HookWidget {
  const CardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    useEffect(() {
      context.read<CardBloc>().add(const CardListFetched());
      return null;
    }, const []);

    return const CardView();
  }
}
```

### 3. BLoC 간 직접 참조

**잘못된 코드:**

```dart
// ❌ BLoC이 다른 BLoC을 직접 참조
class OrderBloc extends Bloc<OrderEvent, OrderState> {
  final CardBloc _cardBloc; // 강한 결합

  Future<void> _onOrderCompleted(OrderCompleted event, Emitter<OrderState> emit) async {
    _cardBloc.add(const CardListFetched()); // 직접 이벤트 발행
  }
}
```

**왜 잘못되었는가:** 순환 의존성과 테스트 복잡도를 유발합니다.

**올바른 방법:**

```dart
// Coordinator — BLoC 간 동기화 담당
@lazySingleton
class CardOrderCoordinator {
  final _controller = StreamController<CardOrderEvent>.broadcast();
  Stream<CardOrderEvent> get events => _controller.stream;

  void notifyOrderCompleted() => _controller.add(CardOrderEvent.orderCompleted);

  @disposeMethod
  void dispose() => _controller.close();
}

// 각 BLoC은 Coordinator의 스트림만 구독
@injectable
class CardBloc extends Bloc<CardEvent, CardState> {
  late final StreamSubscription<CardOrderEvent> _subscription;

  CardBloc(CardOrderCoordinator coordinator, /* ... */)
      : super(const CardState()) {
    _subscription = coordinator.events.listen((event) {
      if (event == CardOrderEvent.orderCompleted) {
        add(const CardListFetched());
      }
    });
    // ...
  }

  @override
  Future<void> close() {
    _subscription.cancel();
    return super.close();
  }
}
```

### 4. State에서 비동기 작업

**잘못된 코드:**

```dart
// ❌ State에서 파생 데이터를 비동기로 계산
class CardState extends Equatable {
  final List<Card> cards;
  Future<int> get totalValue async => // 비동기 getter 금지
      await _calculateTotal(cards);
}
```

**왜 잘못되었는가:** State는 순수 데이터 컨테이너여야 하며, 비동기 로직은 BLoC Handler에서 처리해야 합니다.

**올바른 방법:**

```dart
class CardState extends Equatable {
  final List<Card> cards;
  final int totalValue; // 동기 필드로 저장

  // 단순 computed getter만 허용
  bool get hasCards => cards.isNotEmpty;
  int get cardCount => cards.length;
}
```

### 5. View에서 BLoC guard clause 중복

**잘못된 코드:**

```dart
// ❌ View에서 로딩 중인지 검사 후 이벤트 발행
onPressed: () {
  if (context.read<CardBloc>().state.deleteStatus != AsyncStatus.loading) {
    context.read<CardBloc>().add(CardDeleted(cardId: card.id));
  }
}
```

**왜 잘못되었는가:** Guard clause는 BLoC Handler 책임이며, View에서 중복 검사하면 로직이 분산됩니다.

**올바른 방법:**

```dart
// View — 이벤트만 발행
onPressed: () {
  context.read<CardBloc>().add(CardDeleted(cardId: card.id));
}

// BLoC Handler — guard clause 책임
Future<void> _onCardDeleted(CardDeleted event, Emitter<CardState> emit) async {
  if (state.deleteStatus == AsyncStatus.loading) return; // guard는 여기서만
  // ...
}
```

## 의사결정 기준

### BLoC vs Cubit

| 기준                             | BLoC                            | Cubit                 |
| -------------------------------- | ------------------------------- | --------------------- |
| 이벤트 변환 (debounce, throttle) | ✅ Transformer 지원             | ❌ 불가               |
| 이벤트 추적/로깅                 | ✅ 이벤트 단위 추적             | ❌ 메서드 호출만      |
| 복잡한 비동기 흐름               | ✅ 적합                         | ⚠️ 단순한 것만        |
| 보일러플레이트                   | 많음 (Event + State + Handler)  | 적음 (State + 메서드) |
| 테스트 가독성                    | ✅ 이벤트 기반 시나리오         | ✅ 메서드 호출 기반   |
| 적합한 상태                      | API 호출, 폼 제출, 페이지네이션 | 탭 인덱스, 토글, 필터 |

**의사결정 규칙:** 이벤트 변환(debounce, throttle, droppable)이 필요하거나 복잡한 비동기 플로우가 있으면 BLoC. 단순 상태 토글이나 동기 상태 변경이면 Cubit.

### Event Transformer 선택

> 상세 의사결정 가이드: [BLoC Event Transformer](bloc-event-transformer.md#의사결정-기준)

## 테스트 전략

BLoC 테스트는 **Fake Repository**를 선호합니다. Mockito 대신 직접 구현한 Fake를 사용하면 테스트 의도가 명확해지고 stub 설정이 간단해집니다.

```dart
// Fake Repository
class FakeCardQueryRepository implements CardQueryRepository {
  List<Card> cardsToReturn = [];
  Exception? errorToThrow;

  @override
  Future<Result<List<Card>>> fetchCards() async {
    if (errorToThrow != null) {
      return Result.error(AppError(message: errorToThrow!.toString()));
    }
    return Result.ok(cardsToReturn);
  }
}

// BLoC 테스트
void main() {
  late CardBloc bloc;
  late FakeCardQueryRepository fakeRepository;
  late FetchCardsUseCase fetchCardsUseCase;

  setUp(() {
    fakeRepository = FakeCardQueryRepository();
    fetchCardsUseCase = FetchCardsUseCase(fakeRepository);
    bloc = CardBloc(fetchCardsUseCase);
  });

  tearDown(() => bloc.close());

  blocTest<CardBloc, CardState>(
    '카드 목록 조회 성공 시 cards가 업데이트된다',
    build: () {
      fakeRepository.cardsToReturn = [Card(id: '1', name: 'Test')];
      return bloc;
    },
    act: (bloc) => bloc.add(const CardListFetched()),
    expect: () => [
      const CardState(fetchStatus: AsyncStatus.loading),
      CardState(
        fetchStatus: AsyncStatus.success,
        cards: [Card(id: '1', name: 'Test')],
      ),
    ],
  );

  blocTest<CardBloc, CardState>(
    '카드 목록 조회 실패 시 에러 메시지가 설정된다',
    build: () {
      fakeRepository.errorToThrow = Exception('네트워크 오류');
      return bloc;
    },
    act: (bloc) => bloc.add(const CardListFetched()),
    expect: () => [
      const CardState(fetchStatus: AsyncStatus.loading),
      const CardState(
        fetchStatus: AsyncStatus.error,
        errorMessage: 'Exception: 네트워크 오류',
      ),
    ],
  );
}
```

### Mock 규칙

- **Mock O**: UseCase (Repository 레이어 격리 시)
- **Mock X**: Repository(Fake 사용), BLoC 자체(`blocTest` 사용), Event Transformer(실제 동작 포함 테스트)

## AI가 자주 하는 실수

### 1. Event Transformer 없이 기본 concurrent 사용

모든 핸들러에 Transformer 명시 필수. 상세: [BLoC Event Transformer — AI 실수](bloc-event-transformer.md#ai가-자주-하는-실수)

### 2. Event를 `await`하거나 결과를 기대

**AI가 생성하는 코드:**

```dart
// ❌ bloc.add()의 결과를 await
onPressed: () async {
  await context.read<CardBloc>().add(CardDeleted(cardId: id)); // add()는 void 반환
  ScaffoldMessenger.of(context).showSnackBar(/* ... */);
}
```

**무엇이 잘못되었는가:** `bloc.add()`는 `void` 반환 fire-and-forget이므로 await해도 처리 완료를 기다리지 않습니다.

**올바른 접근법:**

```dart
// BlocListener로 상태 변화에 반응
BlocListener<CardBloc, CardState>(
  listenWhen: (prev, curr) => prev.deleteStatus != curr.deleteStatus,
  listener: (context, state) {
    if (state.deleteStatus == AsyncStatus.success) {
      ScaffoldMessenger.of(context).showSnackBar(/* ... */);
    }
  },
  child: ElevatedButton(
    onPressed: () {
      context.read<CardBloc>().add(CardDeleted(cardId: id)); // fire-and-forget
    },
    child: const Text('삭제'),
  ),
)
```

### 3. copyWith에서 nullable 필드를 null로 초기화하지 못함

**AI가 생성하는 코드:**

```dart
CardState copyWith({
  String? errorMessage,
}) {
  return CardState(
    errorMessage: errorMessage ?? this.errorMessage, // null 전달 불가능
  );
}

// 사용 시 — 에러 메시지 초기화 의도
emit(state.copyWith(errorMessage: null)); // 작동 안 함, 기존 값 유지됨
```

**무엇이 잘못되었는가:** `null ?? this.errorMessage`는 기존 값을 반환하므로 nullable 필드를 null로 되돌릴 수 없습니다.

**올바른 접근법:**

```dart
CardState copyWith({
  String? errorMessage,
  bool clearErrorMessage = false, // 명시적 초기화 플래그
}) {
  return CardState(
    errorMessage: clearErrorMessage ? null : errorMessage ?? this.errorMessage,
  );
}

// 사용 — 명시적 초기화
emit(state.copyWith(clearErrorMessage: true));
```

## 참고 자료

- [flutter_bloc 패키지](https://pub.dev/packages/flutter_bloc) — BLoC/Cubit 공식 패키지
- [bloc_concurrency 패키지](https://pub.dev/packages/bloc_concurrency) — Event Transformer (droppable, sequential, restartable)
- [BLoC 공식 문서](https://bloclibrary.dev/) — 아키텍처 가이드 및 예제
- [stream_transform 패키지](https://pub.dev/packages/stream_transform) — throttle, debounce 등 스트림 변환 유틸리티
