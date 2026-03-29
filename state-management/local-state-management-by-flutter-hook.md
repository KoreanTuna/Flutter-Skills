# Flutter Hook을 활용한 로컬 상태 관리

## 개요

`flutter_hooks` 패키지를 사용하여 Widget 내부의 로컬 상태를 선언적으로 관리하는 패턴을 정의합니다. StatefulWidget의 생명주기 보일러플레이트(`initState` / `dispose`)를 제거하고, 상태 로직을 재사용 가능한 Hook 단위로 추출하여 코드 중복을 줄입니다.

## 사용 시점

- 애니메이션 컨트롤러, TextEditingController, ScrollController 등 컨트롤러 생명주기 관리가 필요할 때
- 탭 인덱스, 토글, 폼 입력값 등 UI 전용 로컬 상태를 다룰 때
- 동일한 상태 로직(예: debounced 검색 입력)이 여러 Widget에서 반복될 때

## 사용하지 말아야 할 때

- 비즈니스 로직이 포함된 상태 → 별도 상태 관리 도구(BLoC, Riverpod 등)로 관리
- 여러 Widget이 공유하는 상태 → 상태 관리 도구 또는 InheritedWidget으로 관리
- 서버 데이터 페칭/캐싱 → 상태 관리 도구에 위임
- 단순한 `const` Widget이나 상태가 전혀 없는 Widget → StatelessWidget 유지

**경계 판단 원칙:** Hook에서 Repository, API Client, UseCase를 직접 호출하는 코드가 보이면 Hook의 역할 범위를 넘은 것입니다.

## 필수 구현

### 기본 사용법: HookWidget

```dart
class SearchBarView extends HookWidget {
  const SearchBarView({super.key});

  @override
  Widget build(BuildContext context) {
    final isExpanded = useState(false);             // 단순 로컬 상태
    final controller = useTextEditingController();  // 컨트롤러 자동 생성/해제
    final focusNode = useFocusNode();               // FocusNode 자동 생성/해제

    return SearchBar(
      controller: controller,
      focusNode: focusNode,
      leading: IconButton(
        icon: Icon(isExpanded.value ? Icons.close : Icons.search),
        onPressed: () => isExpanded.value = !isExpanded.value,
      ),
    );
  }
}
```

### 주요 내장 Hook

| Hook                          | 대체하는 StatefulWidget 코드                      | 용도                        |
| ----------------------------- | ------------------------------------------------- | --------------------------- |
| `useState<T>`                 | `setState` + 필드 변수                             | 단순 로컬 상태              |
| `useTextEditingController`    | `late final` + `initState` + `dispose`             | 텍스트 입력 컨트롤러        |
| `useAnimationController`      | `late final` + `SingleTickerProviderStateMixin`    | 애니메이션 컨트롤러         |
| `useScrollController`         | `late final` + `dispose`                           | 스크롤 컨트롤러             |
| `useFocusNode`                | `late final` + `dispose`                           | 포커스 노드                 |
| `useTabController`            | `late final` + `SingleTickerProviderStateMixin`    | 탭 컨트롤러                 |
| `usePageController`           | `late final` + `dispose`                           | 페이지 컨트롤러             |
| `useEffect`                   | `initState` + `dispose`                            | 부수효과 실행 및 정리       |
| `useMemoized`                 | `late final` (initState에서 한 번 계산)            | 비용이 큰 객체 생성 캐싱    |
| `useValueChanged`             | `didUpdateWidget` 내 비교 로직                     | 값 변경 감지 시 콜백 실행   |
| `useStream`                   | `StreamSubscription` + `initState` + `dispose`     | Stream 구독                 |
| `useFuture`                   | `FutureBuilder` 대체                               | Future 결과 구독            |
| `useValueNotifier`            | `ValueNotifier` + `dispose`                        | ValueNotifier 생성/해제     |
| `useValueListenable`          | `ValueListenableBuilder` 대체                      | ValueListenable 값 구독     |

### useEffect — 부수효과 관리

```dart
class FadeInView extends HookWidget {
  const FadeInView({required this.child, super.key});
  final Widget child;

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: const Duration(milliseconds: 300),
    );

    useEffect(() {
      controller.forward();
      return null; // cleanup 콜백 (없으면 null)
    }, const []); // 빈 배열 = 최초 1회만 실행

    return FadeTransition(opacity: controller, child: child);
  }
}
```

**`keys` 파라미터 동작 규칙:**

| keys 값          | 동작                            | StatefulWidget 대응        |
| ----------------- | ------------------------------- | -------------------------- |
| `const []`        | 최초 1회만 실행                 | `initState`                |
| `[dep1, dep2]`    | 의존값 변경 시마다 실행          | `didUpdateWidget` 내 비교  |
| 생략 (null)       | 매 빌드마다 실행                | `build` 안에서 직접 실행   |

### 커스텀 Hook — 함수 방식

```dart
// debounce가 적용된 텍스트 입력 Hook
String useDebouncedText({
  required TextEditingController controller,
  Duration duration = const Duration(milliseconds: 300),
}) {
  final debouncedValue = useState(controller.text);

  useEffect(() {
    final timer = Timer(duration, () {
      debouncedValue.value = controller.text;
    });
    return timer.cancel; // cleanup — 이전 타이머 취소
  }, [controller.text]);

  return debouncedValue.value;
}
```

### 커스텀 Hook — 클래스 방식

내장 Hook 조합으로 불가능하고, 자체 생명주기(`initHook`/`dispose`)나 `setState` 직접 호출이 필요할 때 사용합니다.

```dart
class _UsePaginationHook extends Hook<PaginationState> {
  const _UsePaginationHook({required this.pageSize});
  final int pageSize;

  @override
  _UsePaginationHookState createState() => _UsePaginationHookState();
}

class _UsePaginationHookState
    extends HookState<PaginationState, _UsePaginationHook> {
  late final ScrollController _scrollController;
  int _currentPage = 0;

  @override
  void initHook() {
    super.initHook();
    _scrollController = ScrollController()..addListener(_onScroll);
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent * 0.8) {
      setState(() => _currentPage++);
    }
  }

  @override
  PaginationState build(BuildContext context) {
    return PaginationState(
      scrollController: _scrollController,
      currentPage: _currentPage,
      pageSize: hook.pageSize,
    );
  }

  @override
  void dispose() => _scrollController.dispose();
}

// 사용측 API
PaginationState usePagination({int pageSize = 20}) {
  return use(_UsePaginationHook(pageSize: pageSize));
}
```

### 핵심 규칙

1. **Hook은 `build` 최상위에서만 호출** — 조건문, 반복문, 콜백 내부 호출 금지. Hook은 호출 순서로 상태를 추적하므로 순서가 변하면 상태 불일치 발생
2. **HookWidget 또는 StatefulHookWidget 상속 필수** — 일반 StatelessWidget/StatefulWidget에서 Hook 호출 불가
3. **로컬 UI 상태만 Hook으로 관리** — 비즈니스 로직(데이터 페칭, 서버 통신)은 상태 관리 도구에 위임
4. **하나의 Widget에 Hook 선언이 5개를 넘으면 분리 검토** — Hook은 `build` 내부에서만 선언 가능하므로, 많아지면 `build`가 비대해지고 단일 책임이 무너진다. 커스텀 Hook으로 추출하거나 Widget 자체를 분리

## 아키텍처 / 구조

```
lib/
├── feature/{domain}/presentation/
│   ├── view/
│   │   └── card_list/
│   │       ├── card_list_view.dart    # HookWidget — 로컬 UI 상태
│   │       └── widget/
│   │           └── card_item.dart     # 순수 UI (Hook 사용 가능하나 최소화)
│   └── hook/                          # Feature 전용 커스텀 Hook
│       ├── use_debounced_text.dart
│       └── use_pagination.dart
└── shared/
    └── hook/                          # 여러 Feature에서 공유하는 커스텀 Hook
        ├── use_toggle.dart
        └── use_delayed_value.dart
```

- 커스텀 Hook 파일 1개 = Hook 함수 1개 (단일 책임)

## 안티패턴

### 1. 조건부 Hook 호출

```dart
// ❌
Widget build(BuildContext context) {
  if (showSearch) {
    final controller = useTextEditingController(); // 조건부 호출
    return TextField(controller: controller);
  }
  return const SizedBox.shrink();
}

// ✅ 항상 호출 — 사용 여부와 관계없이
Widget build(BuildContext context) {
  final controller = useTextEditingController();
  if (!showSearch) return const SizedBox.shrink();
  return TextField(controller: controller);
}
```

### 2. Hook으로 비즈니스 로직 관리

```dart
// ❌ Repository 직접 호출, 에러 핸들링/캐싱 불가
Widget build(BuildContext context) {
  final orders = useState<List<Order>>([]);
  useEffect(() {
    OrderRepository().fetchOrders().then((result) {
      orders.value = result;
    });
    return null;
  }, const []);
}

// ✅ 비즈니스 데이터는 외부에서 주입, Hook은 UI 전용 상태만
class OrderView extends HookWidget {
  final List<Order> orders;
  const OrderView({required this.orders, super.key});

  @override
  Widget build(BuildContext context) {
    final scrollController = useScrollController(); // UI 전용
    return ListView.builder(
      controller: scrollController,
      itemCount: orders.length,
      itemBuilder: (_, i) => OrderItem(order: orders[i]),
    );
  }
}
```

### 3. useEffect keys 누락 / cleanup 누락

```dart
// ❌ keys 생략 — 매 빌드마다 타이머 생성
useEffect(() {
  final timer = Timer(const Duration(milliseconds: 300), () { /* ... */ });
  return timer.cancel;
}); // keys 없음

// ❌ cleanup 누락 — 메모리 누수
useEffect(() {
  final subscription = stream.listen((data) { value.value = data; });
  return null; // subscription 해제 안 됨
}, const []);

// ✅ keys 명시 + cleanup 반환
useEffect(() {
  final subscription = stream.listen((data) { value.value = data; });
  return subscription.cancel;
}, [stream]); // 의존값 변경 시에만 실행
```

### 4. useState에 거대한 객체 사용

```dart
// ❌ 모든 필드를 하나의 useState에 — 어떤 필드든 변경 시 전체 리빌드
final formState = useState(FormData(
  name: '', email: '', phone: '', address: '',
));

// ✅ 독립적인 상태는 개별로 분리
final nameController = useTextEditingController();
final emailController = useTextEditingController();
final phoneController = useTextEditingController();
```

### 5. build 함수에 Hook 과도하게 나열

```dart
// ❌ Hook 12개 나열 — build 비대화, 단일 책임 위반
Widget build(BuildContext context) {
  final nameController = useTextEditingController();
  final emailController = useTextEditingController();
  final phoneController = useTextEditingController();
  final addressController = useTextEditingController();
  final nameFocus = useFocusNode();
  final emailFocus = useFocusNode();
  final phoneFocus = useFocusNode();
  final addressFocus = useFocusNode();
  final isEditing = useState(false);
  final showErrors = useState(false);
  final animationController = useAnimationController(duration: const Duration(milliseconds: 200));
  final scrollController = useScrollController();
  // ...
}
```

**해결 방법 A — 커스텀 Hook으로 관련 로직 묶기:**

```dart
ProfileFormControllers useProfileFormControllers() {
  return ProfileFormControllers(
    name: useTextEditingController(),
    email: useTextEditingController(),
    phone: useTextEditingController(),
    address: useTextEditingController(),
    nameFocus: useFocusNode(),
    emailFocus: useFocusNode(),
    phoneFocus: useFocusNode(),
    addressFocus: useFocusNode(),
  );
}

// build가 깔끔해짐
Widget build(BuildContext context) {
  final form = useProfileFormControllers();
  final isEditing = useState(false);
  return Scaffold(/* ... */);
}
```

**해결 방법 B — Widget 자체를 분리:**

```dart
// 관심사가 다르면 Widget을 나눈다
Widget build(BuildContext context) {
  final isEditing = useState(false);
  return Column(
    children: [
      const ProfileFormFields(),     // 폼 관련 Hook은 이 Widget 내부
      const ProfileAnimatedHeader(), // 애니메이션 관련 Hook은 이 Widget 내부
    ],
  );
}
```

## 의사결정 기준

### Widget 타입 선택

| 기준                                   | StatelessWidget | HookWidget         | StatefulWidget            |
| -------------------------------------- | --------------- | ------------------ | ------------------------- |
| 상태 없음                              | ✅              | ❌ 불필요          | ❌ 불필요                 |
| 컨트롤러/로컬 상태 필요                | ❌              | ✅                 | ✅ (보일러플레이트 많음)  |
| 동일 상태 로직 재사용 필요             | ❌              | ✅ 커스텀 Hook     | ❌ Mixin으로 제한적       |
| Mixin 필요 (WidgetsBindingObserver 등) | ❌              | StatefulHookWidget | ✅                        |
| GlobalKey / State 직접 접근 필요       | ❌              | StatefulHookWidget | ✅                        |

**의사결정 규칙:** 상태 없으면 StatelessWidget. 로컬 UI 상태/컨트롤러 필요하면 HookWidget. Mixin이나 State 직접 접근이 필요하면 StatefulWidget (또는 StatefulHookWidget).

## 테스트 전략

### Widget 테스트

```dart
testWidgets('SearchBarView toggles expansion on icon tap', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(home: Scaffold(body: SearchBarView())),
  );

  expect(find.byIcon(Icons.search), findsOneWidget);
  await tester.tap(find.byType(IconButton));
  await tester.pump();
  expect(find.byIcon(Icons.close), findsOneWidget);
});
```

### 커스텀 Hook 테스트

`HookBuilder`를 사용하여 Hook을 격리 테스트합니다.

```dart
testWidgets('useDebouncedText debounces input', (tester) async {
  late String capturedValue;

  await tester.pumpWidget(
    MaterialApp(
      home: HookBuilder(builder: (context) {
        final controller = useTextEditingController();
        capturedValue = useDebouncedText(
          controller: controller,
          duration: const Duration(milliseconds: 300),
        );
        return TextField(controller: controller);
      }),
    ),
  );

  await tester.enterText(find.byType(TextField), 'hello');
  await tester.pump();
  expect(capturedValue, '');

  await tester.pump(const Duration(milliseconds: 300));
  expect(capturedValue, 'hello');
});
```

- Hook 자체를 Mock하지 않음 — Widget 테스트로 동작 검증
- 컨트롤러는 실제 인스턴스 사용

## 네이밍 컨벤션

| 종류               | 파일 패턴                 | 함수/클래스 패턴      |
| ------------------ | ------------------------- | --------------------- |
| 커스텀 Hook 함수   | `use_{hook_name}.dart`    | `use{HookName}`       |
| 커스텀 Hook 클래스 | `use_{hook_name}.dart`    | `_Use{HookName}Hook`  |
| Hook 결과 타입     | `{hook_name}_state.dart`  | `{HookName}State`     |
| HookWidget         | `{feature}_view.dart`     | `{Feature}View`       |

## 참고 자료

- [flutter_hooks 패키지](https://pub.dev/packages/flutter_hooks) — Hook 기본 API 및 내장 Hook 목록
- [React Hooks 규칙](https://react.dev/reference/rules/rules-of-hooks) — flutter_hooks가 따르는 동일한 규칙 (호출 순서 보장)
- [flutter_hooks GitHub](https://github.com/rrousselGit/flutter_hooks) — 커스텀 Hook 구현 예시
