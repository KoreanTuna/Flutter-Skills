# RepaintBoundary

## Overview

`RepaintBoundary`는 위젯 트리에서 리페인트(repaint) 영역을 격리하여 불필요한 재페인팅을 방지하는 Flutter 성능 최적화 위젯이다. 자식 위젯의 페인팅을 별도의 레이어(layer)로 분리하여, 해당 영역만 독립적으로 다시 그릴 수 있게 한다.

## When To Use

- 애니메이션이 포함된 위젯이 주변 정적 위젯까지 리페인트시키는 경우
- `ListView`, `GridView` 등 스크롤 가능한 리스트의 개별 아이템이 복잡한 경우
- DevTools의 "Highlight Repaints" 기능으로 불필요한 리페인트가 확인된 경우
- Canvas 기반 커스텀 페인팅(`CustomPaint`)이 빈번하게 갱신되는 경우
- 화면 일부만 주기적으로 변경되고 나머지는 정적인 레이아웃

## When NOT To Use

- 위젯이 이미 충분히 가볍고 리페인트 비용이 무시할 수 있는 경우 → 오히려 레이어 합성 오버헤드 발생
- 부모와 자식이 항상 함께 리페인트되는 구조 → 상태 관리 최적화(`const` 위젯, `Selector`, `BlocBuilder` 등)로 해결
- 레이아웃(layout) 자체가 문제인 경우 → `const` 생성자, 위젯 분리, `shouldRebuild` 등 빌드 최적화로 해결
- 단순히 "성능이 느린 것 같아서" 무분별하게 감싸는 경우 → 반드시 프로파일링 후 적용

## Required Implementation

### 기본 사용법

```dart
class DashboardScreen extends StatelessWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 정적 헤더 — 애니메이션 영역의 리페인트로부터 격리
        const RepaintBoundary(
          child: DashboardHeader(),
        ),
        // 빈번하게 갱신되는 차트 애니메이션
        RepaintBoundary(
          child: AnimatedChart(data: chartData),
        ),
        // 정적 푸터
        const RepaintBoundary(
          child: DashboardFooter(),
        ),
      ],
    );
  }
}
```

### ListView 아이템에 적용

```dart
ListView.builder(
  itemCount: items.length,
  // addRepaintBoundaries는 기본값이 true — ListView가 자동으로 각 아이템을 감싼다.
  // 커스텀 제어가 필요한 경우에만 false로 설정 후 직접 관리한다.
  addRepaintBoundaries: true, // default
  itemBuilder: (context, index) {
    return ExpensiveListItem(item: items[index]);
  },
);
```

### CustomPaint와 함께 사용

```dart
class LiveGraphWidget extends StatelessWidget {
  final List<double> dataPoints;

  const LiveGraphWidget({super.key, required this.dataPoints});

  @override
  Widget build(BuildContext context) {
    // CustomPaint는 데이터가 바뀔 때마다 리페인트된다.
    // RepaintBoundary로 감싸서 주변 위젯에 영향을 주지 않게 한다.
    return RepaintBoundary(
      child: CustomPaint(
        painter: GraphPainter(dataPoints),
        size: const Size(double.infinity, 200),
      ),
    );
  }
}
```

### Key Rules

1. **프로파일링 먼저, 최적화는 나중에** — DevTools의 "Highlight Repaints"로 실제 리페인트 범위를 확인한 후에만 적용한다
2. **`ListView`/`GridView`는 기본적으로 `addRepaintBoundaries: true`** — 수동으로 아이템마다 감쌀 필요 없다
3. **`const` 생성자와 함께 사용** — `const RepaintBoundary(child: ...)` 형태로 쓰면 리빌드 자체도 방지된다
4. **중첩 금지** — RepaintBoundary 안에 또 RepaintBoundary를 넣으면 레이어만 늘어나고 효과 없다
5. **레이어 수 관리** — RepaintBoundary 하나당 별도 레이어가 생성되므로, 과도하면 GPU 메모리와 합성 비용이 증가한다

## Architecture / Structure

RepaintBoundary는 별도의 아키텍처를 요구하지 않는다. 기존 위젯 트리에 선택적으로 삽입한다.

```
screen/
├── widgets/
│   ├── static_header.dart       ← const + RepaintBoundary
│   ├── animated_section.dart    ← RepaintBoundary로 격리
│   └── heavy_custom_paint.dart  ← RepaintBoundary로 격리
└── screen.dart
```

**리페인트 격리 판단 기준:**

```
위젯이 자주 리페인트되는가?
├── Yes → 주변 위젯도 함께 리페인트되는가?
│   ├── Yes → RepaintBoundary 적용
│   └── No  → 불필요 (이미 격리됨)
└── No  → 불필요
```

## Anti-Patterns

### 1. 모든 위젯에 무분별하게 감싸기

**What it looks like:**

```dart
// 프로파일링 없이 "혹시 모르니까" 전부 감싸는 패턴
Column(
  children: [
    RepaintBoundary(child: Text('Hello')),
    RepaintBoundary(child: Icon(Icons.star)),
    RepaintBoundary(child: SizedBox(height: 8)),
    RepaintBoundary(child: Text('World')),
  ],
)
```

**Why it's wrong:** 가벼운 위젯까지 감싸면 레이어 합성 비용이 리페인트 절감 효과를 초과한다.

**Do this instead:**

```dart
// 실제로 리페인트가 빈번한 위젯만 격리한다
Column(
  children: [
    const Text('Hello'),
    const Icon(Icons.star),
    const SizedBox(height: 8),
    // 이 위젯만 애니메이션으로 자주 리페인트됨
    RepaintBoundary(
      child: PulsingIndicator(),
    ),
  ],
)
```

### 2. ListView 아이템에 수동으로 중복 적용

**What it looks like:**

```dart
ListView.builder(
  // addRepaintBoundaries 기본값이 true인데도 수동으로 감싸는 패턴
  addRepaintBoundaries: true,
  itemBuilder: (context, index) {
    return RepaintBoundary( // 불필요한 중복
      child: ListTile(title: Text(items[index].name)),
    );
  },
)
```

**Why it's wrong:** `ListView.builder`는 기본적으로 각 아이템을 `RepaintBoundary`로 감싸므로 수동 래핑은 레이어 이중 생성만 유발한다.

**Do this instead:**

```dart
// ListView의 기본 동작을 신뢰한다
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return ListTile(title: Text(items[index].name));
  },
)
```

### 3. 리빌드 문제를 RepaintBoundary로 해결하려는 시도

**What it looks like:**

```dart
class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // setState로 전체가 리빌드되는데 RepaintBoundary로 해결하려 함
        RepaintBoundary(child: ExpensiveWidget()),
        ElevatedButton(
          onPressed: () => setState(() => count++),
          child: Text('Count: $count'),
        ),
      ],
    );
  }
}
```

**Why it's wrong:** `RepaintBoundary`는 리페인트만 격리하며, `setState`로 인한 리빌드는 막지 못한다.

**Do this instead:**

```dart
class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // const로 리빌드 자체를 방지
        const ExpensiveWidget(),
        ElevatedButton(
          onPressed: () => setState(() => count++),
          child: Text('Count: $count'),
        ),
      ],
    );
  }
}
```

## Decision Criteria

| 기준 | RepaintBoundary 적용 | 빌드 최적화 (const, Selector 등) |
|---|---|---|
| 문제 유형 | 불필요한 리페인트 확산 | 불필요한 리빌드 |
| 확인 방법 | DevTools → Highlight Repaints | DevTools → Widget Rebuild Stats |
| 동작 원리 | 페인팅을 별도 레이어로 분리 | 빌드 자체를 스킵 |
| 부작용 | 레이어 수 증가, GPU 메모리 사용 | 없음 (순수 최적화) |
| 적용 순서 | 빌드 최적화 후에도 리페인트가 문제일 때 | 항상 먼저 적용 |

**Decision rule:** 빌드 최적화(`const`, `Selector`, `shouldRebuild`)를 먼저 적용한다. 그래도 DevTools에서 불필요한 리페인트가 확인되면 `RepaintBoundary`를 적용한다.

## Testing Strategy

RepaintBoundary는 렌더링 레벨 최적화이므로 단위 테스트보다는 프로파일링 기반으로 검증한다.

### 프로파일링 기반 검증

```dart
// DevTools에서 확인하는 것이 가장 정확하지만,
// 테스트 코드에서 레이어 존재를 검증할 수 있다.
testWidgets('RepaintBoundary creates separate layer', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(
      home: RepaintBoundary(
        child: Text('Isolated'),
      ),
    ),
  );

  // RenderRepaintBoundary가 위젯 트리에 존재하는지 확인
  final renderObject = tester.renderObject<RenderRepaintBoundary>(
    find.byType(RepaintBoundary),
  );
  expect(renderObject, isNotNull);
  expect(renderObject.debugNeedsPaint, isFalse);
});
```

### 리페인트 횟수 검증

```dart
testWidgets('child repaint does not affect sibling', (tester) async {
  final repaintKey = GlobalKey();

  await tester.pumpWidget(
    MaterialApp(
      home: Column(
        children: [
          RepaintBoundary(
            key: repaintKey,
            child: const StaticWidget(),
          ),
          AnimatedWidget(), // 이 위젯이 리페인트를 트리거
        ],
      ),
    ),
  );

  final boundary = repaintKey.currentContext!.findRenderObject()
      as RenderRepaintBoundary;

  // 리페인트 메트릭 확인 (디버그 모드에서만 사용 가능)
  debugPrint('Repaint count: ${boundary.debugSymmetricPaintCount}');
});
```

### What to Mock

- 별도로 모킹할 대상 없음 — RepaintBoundary는 프레임워크 내장 위젯

### What NOT to Mock

- 렌더링 파이프라인 — 실제 렌더링 동작을 검증해야 하므로 `pumpWidget`으로 실제 렌더 트리를 생성

## Common AI Mistakes

### 1. 리빌드와 리페인트를 혼동하여 RepaintBoundary를 권장

**What AI generates:**

```dart
// "성능 최적화를 위해 RepaintBoundary로 감싸세요"
BlocBuilder<CounterBloc, int>(
  builder: (context, count) {
    return RepaintBoundary( // 잘못된 적용
      child: Column(
        children: [
          Text('Count: $count'),
          const HeavyChart(),
        ],
      ),
    );
  },
)
```

**What's wrong:** `BlocBuilder`가 전체 `builder`를 리빌드하므로 `RepaintBoundary`는 무의미하다.

**Correct approach:**

```dart
// BlocBuilder의 buildWhen으로 리빌드를 제어하고,
// 정적 위젯은 BlocBuilder 바깥으로 분리한다.
Column(
  children: [
    BlocBuilder<CounterBloc, int>(
      builder: (context, count) {
        return Text('Count: $count');
      },
    ),
    // 상태 변경과 무관한 위젯은 밖으로 빼서 리빌드 자체를 방지
    const HeavyChart(),
  ],
)
```

### 2. addRepaintBoundaries를 무시하고 수동 래핑

**What AI generates:**

```dart
// AI가 ListView.builder의 기본 동작을 모르고 수동으로 감싸는 패턴
ListView.builder(
  itemCount: 1000,
  itemBuilder: (context, index) {
    return RepaintBoundary(
      key: ValueKey('repaint_$index'),
      child: Card(
        child: ListTile(
          leading: CircleAvatar(child: Text('${index + 1}')),
          title: Text(items[index].title),
          subtitle: Text(items[index].description),
        ),
      ),
    );
  },
)
```

**What's wrong:** `ListView.builder`의 `addRepaintBoundaries`가 기본 `true`이므로 수동 래핑은 레이어 이중 생성만 유발한다.

**Correct approach:**

```dart
// ListView.builder의 기본 동작을 활용한다
ListView.builder(
  itemCount: 1000,
  itemBuilder: (context, index) {
    return Card(
      child: ListTile(
        leading: CircleAvatar(child: Text('${index + 1}')),
        title: Text(items[index].title),
        subtitle: Text(items[index].description),
      ),
    );
  },
)
```

### 3. RepaintBoundary를 애니메이션 성능 만능 해결책으로 사용

**What AI generates:**

```dart
// "애니메이션이 느리면 RepaintBoundary를 추가하세요"
RepaintBoundary(
  child: AnimatedBuilder(
    animation: _controller,
    builder: (context, child) {
      return Transform.rotate(
        angle: _controller.value * 2 * pi,
        child: child,
      );
    },
    child: const HeavyWidget(), // 100개 이상의 자식 위젯 포함
  ),
)
```

**What's wrong:** 레이어 자체의 페인팅 비용이 크면 격리해도 프레임 드롭은 여전히 발생한다.

**Correct approach:**

```dart
// 1. 먼저 위젯 복잡도를 줄인다
// 2. 그 후 RepaintBoundary로 격리한다
RepaintBoundary(
  child: AnimatedBuilder(
    animation: _controller,
    builder: (context, child) {
      return Transform.rotate(
        angle: _controller.value * 2 * pi,
        child: child,
      );
    },
    child: const SimplifiedWidget(), // 경량화된 위젯
  ),
)
```

## References

- [RepaintBoundary class](https://api.flutter.dev/flutter/widgets/RepaintBoundary-class.html) — 공식 API 문서
- [Flutter rendering pipeline](https://docs.flutter.dev/resources/architectural-overview#rendering-and-layout) — Flutter 렌더링 아키텍처 개요
- [DevTools Performance View](https://docs.flutter.dev/tools/devtools/performance) — 리페인트 프로파일링 도구
- [Flutter internals: RepaintBoundary](https://www.youtube.com/watch?v=UUfXWzp0-DU) — Flutter 렌더링 내부 동작 설명
