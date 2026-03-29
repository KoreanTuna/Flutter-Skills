# BLoC 적용 레이어드 아키텍처

## 개요

Feature 기반 Clean Architecture에서 BLoC을 상태 관리로 채택했을 때의 전체 레이어 구조, 각 레이어의 책임, 레이어 간 의존 방향, 그리고 Presentation 레이어 내부의 Screen → View → Widget 계층 구조를 정의합니다.

## 사용 시점

- BLoC/Cubit을 상태 관리로 선택한 Flutter 프로젝트
- Feature 단위로 모듈화된 중규모 이상 프로젝트
- 팀 규모가 2명 이상이고 레이어별 책임 분리가 필요할 때

## 사용하지 말아야 할 때

- 단순 프로토타입이나 1~2 화면 규모 → 과도한 레이어 분리는 오버헤드
- 상태 관리를 Provider/Riverpod으로 사용 중 → 해당 가이드 참조
- 백엔드 없이 로컬 데이터만 다루는 앱 → Repository 레이어 생략 가능

## 필수 구현

### 전체 레이어 구조

```
lib/feature/{domain}/
├── data/                    # 외부 세계와의 통신
│   ├── datasource/          # API 클라이언트 (Retrofit, gRPC 등)
│   ├── dto/
│   │   ├── request/         # *_request_dto.dart (Freezed + JsonSerializable)
│   │   └── response/        # *_response_dto.dart (Freezed + JsonSerializable)
│   ├── mapper/              # DTO ↔ Domain 변환 (@injectable)
│   └── repository/          # Repository 구현체 (*_impl.dart)
│
├── domain/                  # 순수 비즈니스 로직 (외부 의존 없음)
│   ├── model/               # 도메인 모델 (순수 Freezed, JSON 없음)
│   ├── result/              # Command 결과 객체
│   ├── enum/                # 도메인 열거형
│   ├── repository/          # Repository 인터페이스
│   ├── usecase/             # UseCase
│   └── service/             # Coordinator 등 크로스 BLoC 서비스
│
└── presentation/            # UI + 상태 관리
    ├── bloc/{bloc_name}/    # BLoC + Event + State (part 파일)
    ├── view/{view_name}/    # View + 전용 위젯
    ├── widget/              # Feature 공유 위젯 (순수 UI)
    └── {feature}_screen.dart
```

### 의존 방향 (핵심 규칙)

```
Presentation → Domain ← Data
```

```dart
// Domain 레이어는 어떤 외부 패키지도 import하지 않음
// ✅ domain/model/user.dart
@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required DateTime createdAt, // DateTime 사용 (String 포맷 금지)
  }) = _User;
}

// ❌ domain/model/user.dart — 절대 금지
import 'package:json_annotation/json_annotation.dart'; // Data 레이어 전용
```

### 핵심 규칙

1. **Domain 레이어는 순수해야 한다** — `@JsonSerializable`, `@JsonKey`, HTTP 관련 타입 import 금지
2. **의존 방향은 항상 Domain을 향한다** — Data와 Presentation은 Domain에 의존하고, 서로를 모름
3. **DTO와 Domain Model은 반드시 분리** — Mapper를 통해 변환
4. **Repository는 인터페이스(Domain)와 구현(Data)을 분리** — DI로 연결

## 아키텍처 / 구조

### 단일 화면 Feature

```
lib/feature/card/
├── data/
│   ├── datasource/
│   │   └── card_api.dart
│   ├── dto/
│   │   ├── request/
│   │   │   └── create_card_request_dto.dart
│   │   └── response/
│   │       └── card_response_dto.dart
│   ├── mapper/
│   │   └── card_mapper.dart
│   └── repository/
│       └── card_repository_impl.dart
├── domain/
│   ├── model/
│   │   └── card.dart
│   ├── result/
│   │   └── create_card_result.dart
│   ├── repository/
│   │   └── card_repository.dart
│   └── usecase/
│       └── fetch_cards_usecase.dart
└── presentation/
    ├── bloc/card_list/
    │   ├── card_list_bloc.dart
    │   ├── card_list_event.dart
    │   └── card_list_state.dart
    ├── view/card_list/
    │   ├── card_list_view.dart
    │   └── widget/
    │       └── card_item.dart
    ├── widget/
    │   └── card_status_badge.dart
    └── card_screen.dart
```

### 멀티 화면 Feature (Sub-Feature 패턴)

화면이 많은 Feature는 sub-feature 단위로 colocate:

```
lib/feature/card/presentation/
├── card_list/                        # Sub-feature A
│   ├── bloc/card_list/
│   ├── view/
│   ├── widget/
│   └── card_list_screen.dart
├── card_detail/                      # Sub-feature B
│   ├── bloc/card_detail/
│   ├── view/
│   └── card_detail_screen.dart
├── bloc/card_shared/                 # 여러 sub-feature 공유 BLoC
├── widget/                           # Feature 전체 공유 Widget
├── model/                            # Feature 공유 Presentation Model
├── routes/                           # Feature 라우트 정의
└── card_screen.dart                  # 메인 화면 Screen
```

**배치 규칙:**

- Sub-feature 전용 BLoC → 해당 sub-feature 디렉토리 안에 colocate
- 여러 sub-feature에서 공유하는 BLoC → `presentation/bloc/`에 배치
- `routes/`, `model/`, `widget/`은 feature 전체 공유이므로 presentation 루트에 배치

### DI 등록 규칙 (BLoC 관련)

| 레이어         | Annotation                          | 생명주기  | 이유                 |
| -------------- | ----------------------------------- | --------- | -------------------- |
| BLoC           | `@injectable`                       | Factory   | 화면마다 새 인스턴스 |
| Cubit (글로벌) | `@lazySingleton`                    | Singleton | 앱 전역 공유         |
| Coordinator    | `@lazySingleton` + `@disposeMethod` | Singleton | 스트림 자원 해제     |

### Presentation 레이어: Screen → View → Widget 계층

| 계층       | 역할                            | BLoC 접근                                                                                       |
| ---------- | ------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Screen** | 라우트 진입점, 초기 이벤트 발행 | `initState`(또는 동등한 초기화)에서 이벤트 발행만. **BlocBuilder/Listener/context.select 금지** |
| **View**   | 비즈니스 로직 소비, UI 구성     | `context.select`, `BlocBuilder`, `BlocListener`                                                 |
| **Widget** | 순수 UI 컴포넌트                | **BLoC 접근 금지** — params/callbacks만                                                         |

```dart
// Screen — 라우트 진입점
class CardScreen extends StatefulWidget {
  const CardScreen({super.key});

  @override
  State<CardScreen> createState() => _CardScreenState();
}

class _CardScreenState extends State<CardScreen> {
  @override
  void initState() {
    super.initState();
    context.read<CardListBloc>().add(const CardListDataFetched());
  }

  @override
  Widget build(BuildContext context) {
    return const CardListView(); // View에 위임
  }
}

// View — 비즈니스 로직 소비
class CardListView extends StatelessWidget {
  const CardListView({super.key});

  @override
  Widget build(BuildContext context) {
    final cards = context.select(
      (CardListBloc bloc) => bloc.state.cards,
    );

    return ListView.builder(
      itemCount: cards.length,
      itemBuilder: (_, index) => CardItem(
        card: cards[index],
        onTap: () => context.read<CardListBloc>().add(
          CardSelected(id: cards[index].id),
        ),
      ),
    );
  }
}

// Widget — 순수 UI
class CardItem extends StatelessWidget {
  final Card card;
  final VoidCallback onTap;

  const CardItem({required this.card, required this.onTap, super.key});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(card.name),
      onTap: onTap,
    );
  }
}
```

### 초기 이벤트 발행 위치

| 상황                        | 발행 위치                   | 이유          |
| --------------------------- | --------------------------- | ------------- |
| BLoC이 여러 View에서 공유됨 | Screen `initState`          | 중앙에서 조율 |
| BLoC이 단일 View 전용       | View 초기화 시점            | 자체 관리     |
| Route 파라미터 필요         | Screen → View 생성자로 전달 |               |

### BLoC Provision

```dart
// BlocProvider로 제공
BlocProvider(
  create: (_) => locator<CardListBloc>(),
  child: const CardScreen(),
),
```

**BlocProvider `create`에서 `..add(Event())` 초기 이벤트 발행 금지** — Screen `initState`에서 발행합니다.

### BLoC 간 통신: Coordinator 패턴

BLoC은 서로를 직접 참조하지 않고, Coordinator의 Stream을 구독합니다.

```dart
// domain/service/card_coordinator.dart
@lazySingleton
class CardCoordinator {
  final _controller = StreamController<CardCoordinatorEvent>.broadcast();

  Stream<CardCoordinatorEvent> get stream => _controller.stream;

  void cardCreated(Card card) {
    _controller.add(CardCreated(card));
  }

  @disposeMethod
  void dispose() {
    _controller.close();
  }
}
```

## 안티패턴

### 1. BLoC에서 BuildContext 접근

> [BLoC 패턴 — 안티패턴 #1](../state-management/bloc-pattern.md#1-bloc에서-buildcontext-접근) 참조

### 2. BLoC 간 직접 참조

> [BLoC 패턴 — 안티패턴 #3](../state-management/bloc-pattern.md#3-bloc-간-직접-참조) 참조

### 3. View에서 Repository/UseCase 직접 호출

**잘못된 코드:**

```dart
class CardListView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () async {
        final repo = locator<CardQueryRepository>();
        final cards = await repo.fetchCards(); // ❌
      },
    );
  }
}
```

**왜 잘못되었는가:** BLoC을 우회하면 상태 추적, 로딩/에러 처리, 테스트가 모두 불가능합니다.

**올바른 방법:**

```dart
onPressed: () {
  context.read<CardListBloc>().add(const CardListDataFetched());
},
```

### 4. BlocProvider create에서 초기 이벤트 체이닝

> [BLoC 패턴 — 안티패턴 #2](../state-management/bloc-pattern.md#2-route에서-초기-이벤트-발행) 참조

### 5. Screen에서 BlocBuilder/BlocListener 사용

**잘못된 코드:**

```dart
class _CardScreenState extends State<CardScreen> {
  @override
  void initState() {
    super.initState();
    context.read<CardListBloc>().add(const CardListDataFetched());
  }

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CardListBloc, CardListState>( // ❌
      builder: (context, state) => CardListView(cards: state.cards),
    );
  }
}
```

**왜 잘못되었는가:** Screen은 라우트 진입점과 초기 이벤트 발행만 담당하며, 상태 소비는 View의 책임입니다.

**올바른 방법:**

```dart
// Screen — 이벤트 발행 + View 위임만
class _CardScreenState extends State<CardScreen> {
  @override
  void initState() {
    super.initState();
    context.read<CardListBloc>().add(const CardListDataFetched());
  }

  @override
  Widget build(BuildContext context) => const CardListView(); // View에서 BlocBuilder 사용
}
```

### 6. Widget에서 BLoC 직접 접근

**잘못된 코드:**

```dart
class CardItem extends StatelessWidget {
  final String cardId;

  @override
  Widget build(BuildContext context) {
    final card = context.select( // ❌
      (CardListBloc bloc) => bloc.state.cards.firstWhere((c) => c.id == cardId),
    );
    return ListTile(title: Text(card.name));
  }
}
```

**왜 잘못되었는가:** Widget은 순수 UI 컴포넌트여야 하며, BLoC에 의존하면 재사용과 단위 테스트가 불가능합니다.

**올바른 방법:**

```dart
class CardItem extends StatelessWidget {
  final Card card;        // 데이터는 파라미터로
  final VoidCallback onTap; // 액션은 콜백으로

  const CardItem({required this.card, required this.onTap, super.key});

  @override
  Widget build(BuildContext context) {
    return ListTile(title: Text(card.name), onTap: onTap);
  }
}
```

### 7. 초기화 시점에서 BLoC 상태 읽고 조건 분기

**잘못된 코드:**

```dart
@override
void initState() {
  super.initState();
  final state = context.read<CardListBloc>().state;
  if (state.cards.isEmpty) { // ❌ race condition 위험
    context.read<CardListBloc>().add(const CardListDataFetched());
  }
}
```

**왜 잘못되었는가:** 초기화 시점의 상태는 race condition 위험이 있으므로 guard 조건은 BLoC 핸들러 내부에서 처리해야 합니다.

**올바른 방법:**

```dart
// initState — 무조건 이벤트 발행
@override
void initState() {
  super.initState();
  context.read<CardListBloc>().add(const CardListDataFetched());
}

// BLoC 핸들러 — guard clause로 중복 방지
void _onDataFetched(CardListDataFetched event, Emitter<CardListState> emit) {
  if (state.status == AsyncStatus.loading) return; // guard
  // ...
}
```

## 의사결정 기준

### BLoC vs Cubit 선택

> [BLoC 패턴 — BLoC vs Cubit 의사결정](../state-management/bloc-pattern.md#bloc-vs-cubit) 참조

### 단일 화면 vs 멀티 화면 구조 선택

| 기준           | 단일 화면 구조 | 멀티 화면 (Sub-Feature) |
| -------------- | -------------- | ----------------------- |
| 화면 수        | 1~2개          | 3개 이상                |
| BLoC 공유 여부 | 대부분 전용    | 공유 + 전용 혼재        |
| 라우팅 복잡도  | 단순           | 중첩 라우트 필요        |

## 테스트 전략

### BLoC 테스트 — Fake Repository 선호

```dart
// Fake Repository — 실제 동작을 모방
class FakeCardQueryRepository implements CardQueryRepository {
  List<Card> cards = [];

  @override
  Future<Result<List<Card>>> fetchCards(CardCriteria criteria) async {
    return Result.ok(cards);
  }
}

// BLoC 테스트
void main() {
  late CardListBloc bloc;
  late FakeCardQueryRepository fakeRepository;

  setUp(() {
    fakeRepository = FakeCardQueryRepository();
    final useCase = FetchCardsUseCase(fakeRepository);
    bloc = CardListBloc(useCase);
  });

  blocTest<CardListBloc, CardListState>(
    'emits loaded state when cards fetched',
    build: () {
      fakeRepository.cards = [
        const Card(id: '1', name: 'Test Card'),
      ];
      return bloc;
    },
    act: (bloc) => bloc.add(const CardListDataFetched()),
    expect: () => [
      // loading state
      isA<CardListState>().having((s) => s.status, 'status', AsyncStatus.loading),
      // loaded state
      isA<CardListState>()
          .having((s) => s.status, 'status', AsyncStatus.success)
          .having((s) => s.cards.length, 'cards length', 1),
    ],
  );
}
```

### Mock 하지 말아야 하는 것

- **Repository** — Fake 구현체 사용 (Mockito Mock 대신)

## AI가 자주 하는 실수

### 1. BLoC에서 다른 BLoC 직접 주입

BLoC 간 직접 의존은 순환 참조와 테스트 복잡도를 유발합니다. Coordinator 패턴으로 간접 통신해야 합니다.
> [안티패턴 #2](#2-bloc-간-직접-참조) 및 [BLoC 패턴 — 안티패턴 #3](../state-management/bloc-pattern.md#3-bloc-간-직접-참조) 참조

### 2. Screen/View/Widget 계층 무시

**AI가 생성하는 코드:**

```dart
// 하나의 파일에 모든 것을 넣음
class CardScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CardListBloc, CardListState>( // Screen에서 BlocBuilder ❌
      builder: (context, state) {
        return ListView.builder(
          itemBuilder: (_, i) {
            final card = state.cards[i];
            return ListTile(
              title: Text(card.name),
              onTap: () => context.read<CardListBloc>().add(...), // Widget에서 BLoC ❌
            );
          },
        );
      },
    );
  }
}
```

**무엇이 잘못되었는가:** Screen → View → Widget 계층이 없으면 한 파일이 비대해지고, 위젯 리빌드 최적화가 불가능하며, Widget 재사용이 안 됩니다.

**올바른 접근법:** Screen(이벤트 발행만) → View(BLoC 소비, UI 구성) → Widget(순수 UI, params/callbacks) 으로 분리.

### 3. BlocProvider create에서 이벤트 체이닝

`create`에서는 BLoC 생성만 하고, 초기 이벤트는 Screen의 `initState`에서 발행해야 합니다.
> [안티패턴 #4](#4-blocprovider-create에서-초기-이벤트-체이닝) 및 [BLoC 패턴 — 안티패턴 #2](../state-management/bloc-pattern.md#2-route에서-초기-이벤트-발행) 참조

## 네이밍 컨벤션

| 레이어      | 파일 패턴                   | 클래스 패턴                     |
| ----------- | --------------------------- | ------------------------------- |
| BLoC        | `{feature}_bloc.dart`       | `{Feature}Bloc`                 |
| Event       | part of BLoC                | `{Resource}{Action}` (과거분사) |
| State       | part of BLoC                | `{Feature}State`                |
| Coordinator | `{domain}_coordinator.dart` | `{Domain}Coordinator`           |
| Screen      | `{feature}_screen.dart`     | `{Feature}Screen`               |
| View        | `{feature}_view.dart`       | `{Feature}View`                 |

## 참고 자료

- [flutter_bloc 공식 문서](https://bloclibrary.dev/) — BLoC/Cubit 패턴 기본
- [bloc_test 패키지](https://pub.dev/packages/bloc_test) — BLoC 테스트 유틸리티
