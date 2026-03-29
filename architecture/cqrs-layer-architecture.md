# CQRS 적용 레이어드 아키텍처

## 개요

Layer Architecture에 CQRS(Command Query Responsibility Segregation) 패턴을 적용하여 Repository를 Command(쓰기)와 Query(읽기)로 분리합니다. 이를 통해 각 Repository가 단일 책임을 갖게 되고, UseCase의 의도가 명확해지며, BLoC의 역할 분리가 자연스럽게 따라옵니다.

## 사용 시점

- 하나의 Repository에 CRUD 메서드가 5개 이상 모여 비대해질 때
- 읽기와 쓰기의 데이터 소스나 캐싱 전략이 다를 때
- 팀원 간 작업 분리가 필요할 때 (읽기 담당 / 쓰기 담당)
- BLoC이 읽기·쓰기를 모두 처리하여 비대해질 때

## 사용하지 말아야 할 때

- CRUD가 단순하고 메서드가 3개 이하 → 단일 Repository로 충분 (`bloc-apply-layer-architecture.md` 참조)
- 읽기/쓰기가 항상 동일한 데이터 소스 + 동일한 변환 로직 → 분리가 오버헤드
- 프로토타입이나 1~2 화면 규모 → 레이어 분리 자체가 불필요

## 필수 구현

### CQRS 적용 Repository 분리

**기존 단일 Repository (분리 전):**

```dart
// domain/repository/card_repository.dart
abstract interface class CardRepository {
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria);
  Future<Result<Card>> fetchCardById(String id);
  Future<Result<CreateCardResult>> createCard(CreateCardCommand command);
  Future<Result<void>> updateCard(UpdateCardCommand command);
  Future<Result<void>> deleteCard(String id);
  Future<Result<void>> reorderCards(List<String> orderedIds);
}
```

**CQRS 적용 후:**

```dart
// domain/repository/card_query_repository.dart
abstract interface class CardQueryRepository {
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria);
  Future<Result<Card>> fetchCardById(String id);
  Future<Result<List<Card>>> searchCards(String keyword);
}

// domain/repository/card_command_repository.dart
abstract interface class CardCommandRepository {
  Future<Result<CreateCardResult>> createCard(CreateCardCommand command);
  Future<Result<void>> updateCard(UpdateCardCommand command);
  Future<Result<void>> deleteCard(String id);
  Future<Result<void>> reorderCards(List<String> orderedIds);
}
```

### UseCase도 Query/Command로 분리

```dart
// domain/usecase/fetch_cards_usecase.dart — Query
@injectable
class FetchCardsUseCase {
  final CardQueryRepository _repository;

  const FetchCardsUseCase(this._repository);

  Future<Result<List<Card>>> call(CardCriteria criteria) {
    return _repository.fetchCards(criteria);
  }
}

// domain/usecase/create_card_usecase.dart — Command
@injectable
class CreateCardUseCase {
  final CardCommandRepository _repository;

  const CreateCardUseCase(this._repository);

  Future<Result<CreateCardResult>> call(CreateCardCommand command) {
    return _repository.createCard(command);
  }
}
```

### BLoC 분리: Query BLoC + Command BLoC

```dart
// presentation/bloc/card_list/card_list_bloc.dart — Query 전용
@injectable
class CardListBloc extends Bloc<CardListEvent, CardListState> {
  final FetchCardsUseCase _fetchCards;

  CardListBloc(this._fetchCards) : super(const CardListState()) {
    on<CardListDataFetched>(_onDataFetched);
    on<CardListRefreshed>(_onRefreshed);
  }

  Future<void> _onDataFetched(
    CardListDataFetched event,
    Emitter<CardListState> emit,
  ) async {
    if (state.status == AsyncStatus.loading) return;
    emit(state.copyWith(status: AsyncStatus.loading));

    final result = await _fetchCards(event.criteria);
    result.when(
      ok: (cards) => emit(state.copyWith(
        status: AsyncStatus.success,
        cards: cards,
      )),
      error: (failure) => emit(state.copyWith(
        status: AsyncStatus.failure,
        failure: failure,
      )),
    );
  }
}

// presentation/bloc/card_command/card_command_bloc.dart — Command 전용
@injectable
class CardCommandBloc extends Bloc<CardCommandEvent, CardCommandState> {
  final CreateCardUseCase _createCard;
  final UpdateCardUseCase _updateCard;
  final DeleteCardUseCase _deleteCard;
  final CardCoordinator _coordinator;

  CardCommandBloc(
    this._createCard,
    this._updateCard,
    this._deleteCard,
    this._coordinator,
  ) : super(const CardCommandState()) {
    on<CardCreateRequested>(_onCreateRequested);
    on<CardUpdateRequested>(_onUpdateRequested);
    on<CardDeleteRequested>(_onDeleteRequested);
  }

  Future<void> _onCreateRequested(
    CardCreateRequested event,
    Emitter<CardCommandState> emit,
  ) async {
    emit(state.copyWith(status: AsyncStatus.loading));

    final result = await _createCard(event.command);
    result.when(
      ok: (created) {
        emit(state.copyWith(status: AsyncStatus.success));
        _coordinator.cardCreated(created.card); // Query BLoC에 알림
      },
      error: (failure) => emit(state.copyWith(
        status: AsyncStatus.failure,
        failure: failure,
      )),
    );
  }
}
```

### Command → Query 동기화: Coordinator 패턴

Command 성공 후 Query BLoC에 목록 갱신을 알리기 위해 Coordinator를 사용합니다.

> Coordinator 기본 개념: [BLoC 패턴 — BLoC 간 직접 참조](../state-management/bloc-pattern.md#3-bloc-간-직접-참조)

```dart
// domain/service/card_coordinator.dart
@lazySingleton
class CardCoordinator {
  final _controller = StreamController<CardCoordinatorEvent>.broadcast();

  Stream<CardCoordinatorEvent> get stream => _controller.stream;

  void cardCreated(Card card) {
    _controller.add(CardCreated(card));
  }

  void cardUpdated(Card card) {
    _controller.add(CardUpdated(card));
  }

  void cardDeleted(String id) {
    _controller.add(CardDeleted(id));
  }

  @disposeMethod
  void dispose() {
    _controller.close();
  }
}

// Query BLoC에서 Coordinator 구독
@injectable
class CardListBloc extends Bloc<CardListEvent, CardListState> {
  final FetchCardsUseCase _fetchCards;
  late final StreamSubscription _coordinatorSub;

  CardListBloc(this._fetchCards, CardCoordinator coordinator)
      : super(const CardListState()) {
    on<CardListDataFetched>(_onDataFetched);
    on<CardListRefreshed>(_onRefreshed);

    _coordinatorSub = coordinator.stream.listen((event) {
      switch (event) {
        case CardCreated():
        case CardUpdated():
        case CardDeleted():
          add(const CardListRefreshed()); // 목록 재조회
      }
    });
  }

  @override
  Future<void> close() {
    _coordinatorSub.cancel();
    return super.close();
  }
}
```

### Data 레이어 구현체 분리

```dart
// data/repository/card_query_repository_impl.dart
@LazySingleton(as: CardQueryRepository)
class CardQueryRepositoryImpl implements CardQueryRepository {
  final CardApi _api;
  final CardMapper _mapper;

  const CardQueryRepositoryImpl(this._api, this._mapper);

  @override
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria) async {
    try {
      final dtos = await _api.getCards(criteria.toQueryParams());
      return Result.ok(dtos.map(_mapper.toDomain).toList());
    } on DioException catch (e) {
      return Result.error(e.toFailure());
    }
  }
}

// data/repository/card_command_repository_impl.dart
@LazySingleton(as: CardCommandRepository)
class CardCommandRepositoryImpl implements CardCommandRepository {
  final CardApi _api;
  final CardMapper _mapper;

  const CardCommandRepositoryImpl(this._api, this._mapper);

  @override
  Future<Result<CreateCardResult>> createCard(CreateCardCommand command) async {
    try {
      final dto = await _api.createCard(
        _mapper.toCreateRequestDto(command),
      );
      return Result.ok(_mapper.toCreateResult(dto));
    } on DioException catch (e) {
      return Result.error(e.toFailure());
    }
  }
}
```

### 핵심 규칙

1. **Query Repository는 데이터를 변경하지 않는다** — `fetch`, `get`, `search`, `find` 메서드만 포함
2. **Command Repository는 데이터를 반환하지 않거나 최소한의 결과만 반환** — `Result<void>` 또는 `Result<CreateResult>` 수준
3. **Command 성공 후 Query 갱신은 Coordinator를 통해** — Command BLoC이 Query BLoC을 직접 참조하지 않음
4. **UseCase는 하나의 Repository 타입만 의존** — Query UseCase는 QueryRepository만, Command UseCase는 CommandRepository만
5. **BLoC 분리는 선택적** — Repository/UseCase 분리만으로도 CQRS 효과를 얻을 수 있음. BLoC이 비대해질 때 분리

## 아키텍처 / 구조

### CQRS 적용 Feature 폴더 구조

```
lib/feature/card/
├── data/
│   ├── datasource/
│   │   └── card_api.dart
│   ├── dto/
│   │   ├── request/
│   │   │   ├── create_card_request_dto.dart
│   │   │   └── update_card_request_dto.dart
│   │   └── response/
│   │       └── card_response_dto.dart
│   ├── mapper/
│   │   └── card_mapper.dart
│   └── repository/
│       ├── card_query_repository_impl.dart    # Query 구현
│       └── card_command_repository_impl.dart   # Command 구현
│
├── domain/
│   ├── model/
│   │   └── card.dart
│   ├── result/
│   │   └── create_card_result.dart
│   ├── repository/
│   │   ├── card_query_repository.dart          # Query 인터페이스
│   │   └── card_command_repository.dart         # Command 인터페이스
│   ├── usecase/
│   │   ├── fetch_cards_usecase.dart             # Query UseCase
│   │   ├── search_cards_usecase.dart            # Query UseCase
│   │   ├── create_card_usecase.dart             # Command UseCase
│   │   ├── update_card_usecase.dart             # Command UseCase
│   │   └── delete_card_usecase.dart             # Command UseCase
│   └── service/
│       └── card_coordinator.dart
│
└── presentation/
    ├── bloc/
    │   ├── card_list/                           # Query BLoC
    │   │   ├── card_list_bloc.dart
    │   │   ├── card_list_event.dart
    │   │   └── card_list_state.dart
    │   └── card_command/                        # Command BLoC
    │       ├── card_command_bloc.dart
    │       ├── card_command_event.dart
    │       └── card_command_state.dart
    ├── view/card_list/
    │   ├── card_list_view.dart
    │   └── widget/
    │       └── card_item.dart
    ├── widget/
    │   └── card_status_badge.dart
    └── card_screen.dart
```

### 의존 방향

```
Presentation → Domain ← Data

CardListBloc (Query) ──→ FetchCardsUseCase ──→ CardQueryRepository
CardCommandBloc (Command) ──→ CreateCardUseCase ──→ CardCommandRepository
                        └──→ CardCoordinator (이벤트 발행)
CardListBloc ←── CardCoordinator (이벤트 구독, 목록 갱신)
```

### DI 등록 규칙

| 계층 | Annotation | Scope | 이유 |
| --- | --- | --- | --- |
| Query/Command Repo Impl | `@LazySingleton(as: ...)` | Singleton | 캐싱·트랜잭션 공유 |
| Query/Command UseCase | `@injectable` | Factory | 상태 없음 |
| Query/Command BLoC | `@injectable` | Factory | 화면마다 새 인스턴스 |
| Coordinator | `@lazySingleton` + `@disposeMethod` | Singleton | BLoC 간 이벤트 중계 |

### Screen에서 Query/Command BLoC 동시 사용

`MultiBlocProvider`로 두 BLoC을 제공하고, View에서 읽기는 Query BLoC, 쓰기는 Command BLoC으로 분리합니다.

```dart
// MultiBlocProvider로 두 BLoC 제공
MultiBlocProvider(
  providers: [
    BlocProvider(create: (_) => locator<CardListBloc>()),
    BlocProvider(create: (_) => locator<CardCommandBloc>()),
  ],
  child: const CardScreen(),
),

// View — Query BLoC(select)으로 읽기, Command BLoC(read)으로 쓰기
class CardListView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final cards = context.select((CardListBloc b) => b.state.cards);

    return BlocListener<CardCommandBloc, CardCommandState>(
      listenWhen: (prev, curr) => prev.status != curr.status,
      listener: (context, state) { /* 성공/실패 SnackBar */ },
      child: ListView.builder(
        itemCount: cards.length,
        itemBuilder: (_, i) => CardItem(
          card: cards[i],
          onDelete: () => context.read<CardCommandBloc>().add(
            CardDeleteRequested(id: cards[i].id),
          ),
        ),
      ),
    );
  }
}
```

## 안티패턴

### 1. 하나의 Repository에 Query/Command 혼재 (God Repository)

**잘못된 코드:**

```dart
abstract interface class CardRepository {
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria);
  Future<Result<Card>> fetchCardById(String id);
  Future<Result<List<Card>>> searchCards(String keyword);
  Future<Result<CreateCardResult>> createCard(CreateCardCommand cmd);
  Future<Result<void>> updateCard(UpdateCardCommand cmd);
  Future<Result<void>> deleteCard(String id);
  Future<Result<void>> reorderCards(List<String> ids);
  Future<Result<void>> archiveCard(String id);
  Future<Result<void>> duplicateCard(String id);
}
```

**왜 잘못되었는가:** 읽기·쓰기가 섞여 단일 책임 원칙을 위반하고, 캐싱 전략과 트랜잭션 관리를 한 클래스에서 처리해야 합니다.

**올바른 방법:**

```dart
// Query — 읽기만
abstract interface class CardQueryRepository {
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria);
  Future<Result<Card>> fetchCardById(String id);
  Future<Result<List<Card>>> searchCards(String keyword);
}

// Command — 쓰기만
abstract interface class CardCommandRepository {
  Future<Result<CreateCardResult>> createCard(CreateCardCommand cmd);
  Future<Result<void>> updateCard(UpdateCardCommand cmd);
  Future<Result<void>> deleteCard(String id);
  Future<Result<void>> reorderCards(List<String> ids);
  Future<Result<void>> archiveCard(String id);
  Future<Result<void>> duplicateCard(String id);
}
```

### 2. Command UseCase가 Query Repository에 의존

**잘못된 코드:**

```dart
@injectable
class CreateCardUseCase {
  final CardCommandRepository _commandRepo;
  final CardQueryRepository _queryRepo; // ❌ Command가 Query에 의존

  Future<Result<Card>> call(CreateCardCommand command) async {
    final result = await _commandRepo.createCard(command);
    return result.when(
      ok: (created) => _queryRepo.fetchCardById(created.id), // ❌ 생성 후 재조회
      error: Result.error,
    );
  }
}
```

**왜 잘못되었는가:** Command가 Query에 의존하면 CQRS 분리 의미가 없어지며, 재조회는 Query BLoC의 책임입니다.

**올바른 방법:**

```dart
@injectable
class CreateCardUseCase {
  final CardCommandRepository _repository;

  const CreateCardUseCase(this._repository);

  Future<Result<CreateCardResult>> call(CreateCardCommand command) {
    return _repository.createCard(command); // 쓰기만 수행
  }
}

// 목록 갱신은 Coordinator → Query BLoC에서 처리
```

### 3. Command BLoC에서 직접 Query BLoC 상태 갱신

**잘못된 코드:**

```dart
class CardCommandBloc extends Bloc<CardCommandEvent, CardCommandState> {
  final CardListBloc _listBloc; // ❌ Command가 Query BLoC 직접 참조

  Future<void> _onCreateRequested(
    CardCreateRequested event,
    Emitter<CardCommandState> emit,
  ) async {
    final result = await _createCard(event.command);
    result.when(
      ok: (created) {
        _listBloc.add(const CardListRefreshed()); // ❌ 직접 이벤트 발행
      },
      error: (_) {},
    );
  }
}
```

**왜 잘못되었는가:** Command가 Query BLoC에 직접 의존하면 양방향 결합이 생기고 CQRS 분리 원칙을 무너뜨립니다.

**올바른 방법:**

```dart
class CardCommandBloc extends Bloc<CardCommandEvent, CardCommandState> {
  final CardCoordinator _coordinator; // Coordinator만 의존

  Future<void> _onCreateRequested(
    CardCreateRequested event,
    Emitter<CardCommandState> emit,
  ) async {
    final result = await _createCard(event.command);
    result.when(
      ok: (created) {
        _coordinator.cardCreated(created.card); // Coordinator에 알림
      },
      error: (_) {},
    );
  }
}
```

### 4. Query BLoC에 쓰기 이벤트 추가

**잘못된 코드:**

```dart
// Query BLoC에 Command 이벤트가 섞여 있음
sealed class CardListEvent {
  const CardListEvent();
}

class CardListDataFetched extends CardListEvent { ... }
class CardListRefreshed extends CardListEvent { ... }
class CardCreateRequested extends CardListEvent { ... }  // ❌ 쓰기 이벤트
class CardDeleteRequested extends CardListEvent { ... }  // ❌ 쓰기 이벤트
```

**왜 잘못되었는가:** Query BLoC에 쓰기 로직이 섞이면 CQRS 분리가 무의미합니다.

**올바른 방법:**

```dart
// Query BLoC — 읽기 이벤트만
sealed class CardListEvent {
  const CardListEvent();
}

class CardListDataFetched extends CardListEvent {
  final CardCriteria criteria;
  const CardListDataFetched({required this.criteria});
}

class CardListRefreshed extends CardListEvent {
  const CardListRefreshed();
}

// Command BLoC — 쓰기 이벤트만
sealed class CardCommandEvent {
  const CardCommandEvent();
}

class CardCreateRequested extends CardCommandEvent {
  final CreateCardCommand command;
  const CardCreateRequested({required this.command});
}

class CardDeleteRequested extends CardCommandEvent {
  final String id;
  const CardDeleteRequested({required this.id});
}
```

## 의사결정 기준

### 단일 Repository vs CQRS 분리

| 기준                                  | 단일 Repository | CQRS 분리           |
| ------------------------------------- | --------------- | -------------------- |
| Repository 메서드 수                  | 3개 이하        | 4개 이상             |
| 읽기/쓰기 데이터 소스 차이            | 동일            | 다름 (캐시 vs API)   |
| BLoC 비대화                           | 관리 가능       | 읽기/쓰기 분리 필요  |
| 팀 규모                               | 1~2명           | 3명 이상             |
| 읽기/쓰기 캐싱·최적화 전략 차이       | 없음            | 있음                 |

**의사결정 규칙:** Repository 메서드가 4개 이상이고, 읽기와 쓰기의 관심사(캐싱, 데이터 소스, 에러 처리)가 다르다면 CQRS 분리. 그렇지 않으면 단일 Repository로 시작하고 필요 시 분리.

### BLoC 분리 시점

| 기준                              | 단일 BLoC 유지   | Query/Command BLoC 분리 |
| --------------------------------- | ---------------- | ----------------------- |
| BLoC Event 수                     | 4개 이하         | 5개 이상                |
| 읽기/쓰기 상태 관리 복잡도        | 단순             | 각각 독립적 상태 필요   |
| 화면에서 읽기/쓰기 UI가 분리됨    | 같은 화면        | 다른 화면 또는 다이얼로그 |

**의사결정 규칙:** Repository는 CQRS로 분리하되, BLoC은 단일 BLoC으로 시작합니다. Event가 5개 이상이거나 읽기/쓰기 상태가 독립적으로 관리되어야 할 때 BLoC을 분리합니다.

## 테스트 전략

### Query UseCase 테스트

Query UseCase는 단순 위임이므로 Fake Repository에 데이터를 설정하고 결과를 검증합니다.

### Command BLoC 테스트 — Coordinator 발행 검증

```dart
// setUp: 실제 CardCoordinator 인스턴스 + coordinatorEvents 리스트로 이벤트 수집
blocTest<CardCommandBloc, CardCommandState>(
  'emits success and notifies coordinator when card created',
  build: () {
    fakeCommandRepo.createResult = CreateCardResult(
      card: const Card(id: '1', name: 'New Card'),
    );
    return bloc;
  },
  act: (bloc) => bloc.add(CardCreateRequested(
    command: const CreateCardCommand(name: 'New Card'),
  )),
  expect: () => [
    isA<CardCommandState>().having((s) => s.status, 'status', AsyncStatus.loading),
    isA<CardCommandState>().having((s) => s.status, 'status', AsyncStatus.success),
  ],
  verify: (_) {
    expect(coordinatorEvents, hasLength(1));
    expect(coordinatorEvents.first, isA<CardCreated>());
  },
);
```

### Query BLoC 테스트 — Coordinator 구독 검증

```dart
blocTest<CardListBloc, CardListState>(
  'refreshes when coordinator emits CardCreated',
  setUp: () {
    fakeQueryRepo.cards = [
      const Card(id: '1', name: 'Existing'),
      const Card(id: '2', name: 'New Card'),
    ];
  },
  build: () => bloc,
  act: (bloc) => coordinator.cardCreated(
    const Card(id: '2', name: 'New Card'),
  ),
  expect: () => [
    isA<CardListState>().having(
      (s) => s.status, 'status', AsyncStatus.loading,
    ),
    isA<CardListState>()
        .having((s) => s.status, 'status', AsyncStatus.success)
        .having((s) => s.cards.length, 'cards length', 2),
  ],
);
```

### Mock 대상

- **Query Repository**: Fake 구현 — 읽기 데이터를 미리 설정
- **Command Repository**: Fake 구현 — 성공/실패 결과를 미리 설정
- **Coordinator**: 실제 인스턴스 사용 — Stream 동작을 검증해야 함

### Mock 하지 말아야 하는 것

- **Coordinator**: Mockito로 Stream을 모킹하면 실제 이벤트 전파 동작을 검증할 수 없음. 실제 인스턴스 사용
- **UseCase**: BLoC 테스트에서 UseCase를 모킹하면 실제 호출 흐름이 끊어짐. Fake Repository로 간접 제어

## AI가 자주 하는 실수

### 1. CQRS 없이 하나의 BLoC에 모든 것을 넣음

**AI가 생성하는 코드:**

```dart
@injectable
class CardBloc extends Bloc<CardEvent, CardState> {
  final CardRepository _repository; // 단일 Repository

  CardBloc(this._repository) : super(const CardState()) {
    on<CardsFetched>(_onFetched);
    on<CardCreated>(_onCreated);
    on<CardUpdated>(_onUpdated);
    on<CardDeleted>(_onDeleted);
    on<CardsSearched>(_onSearched);
    on<CardReordered>(_onReordered);
    on<CardArchived>(_onArchived);
  }

  // 7개 이벤트 핸들러가 모두 하나의 BLoC에...
}
```

**무엇이 잘못되었는가:** 로딩 상태가 읽기/쓰기 구분이 안 되고 테스트 시 모든 의존성을 제공해야 합니다.

**올바른 접근법:**

```dart
class CardListBloc extends Bloc<CardListEvent, CardListState> { ... }     // 읽기 전용
class CardCommandBloc extends Bloc<CardCommandEvent, CardCommandState> { ... } // 쓰기 전용 + Coordinator 알림
```

### 2. Command 결과를 Query 상태에 직접 반영

**AI가 생성하는 코드:**

```dart
Future<void> _onCreateRequested(
  CardCreateRequested event,
  Emitter<CardCommandState> emit,
) async {
  final result = await _createCard(event.command);
  result.when(
    ok: (created) {
      // ❌ Command BLoC에서 목록을 직접 조작
      final updatedCards = [...state.cards, created.card];
      emit(state.copyWith(cards: updatedCards));
    },
    error: (_) {},
  );
}
```

**무엇이 잘못되었는가:** Command BLoC이 읽기 상태를 직접 조작하면 서버 정합성(정렬, 필터, 페이지네이션)이 깨집니다.

**올바른 접근법:** Command 성공 시 `_coordinator.cardCreated(created.card)`로 알림만 발행하고, Query BLoC이 서버에서 목록을 재조회합니다. (필수 구현 섹션의 Coordinator 패턴 참조)

### 3. Query/Command Repository 구현체를 같은 클래스로 합침

**AI가 생성하는 코드:**

```dart
// ❌ 하나의 구현체가 두 인터페이스를 모두 구현
@LazySingleton(as: CardQueryRepository)
@LazySingleton(as: CardCommandRepository) // DI 충돌
class CardRepositoryImpl implements CardQueryRepository, CardCommandRepository {
  // 모든 메서드가 한 클래스에...
}
```

**무엇이 잘못되었는가:** 구현체가 하나이면 독립적 캐싱·테스트 등 CQRS의 이점을 얻을 수 없고 DI 등록도 복잡해집니다.

**올바른 접근법:** 구현체도 분리합니다. API 클라이언트는 공유 가능하지만 Repository 구현체는 반드시 분리합니다.

## 참고 자료

- [CQRS Pattern — Martin Fowler](https://martinfowler.com/bliki/CQRS.html) — CQRS 패턴 원본 정의
- [flutter_bloc 공식 문서](https://bloclibrary.dev/) — BLoC/Cubit 패턴 기본
- [bloc_test 패키지](https://pub.dev/packages/bloc_test) — BLoC 테스트 유틸리티
