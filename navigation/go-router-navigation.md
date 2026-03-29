# GoRouter 네비게이션

## 개요

`go_router` 패키지를 사용하여 Flutter 앱의 선언적 라우팅을 구현하는 패턴을 정의합니다. URL 기반 네비게이션, 딥링크, 리다이렉트, 중첩 네비게이션(ShellRoute)을 타입 안전하게 관리하며, Navigator 2.0의 복잡성을 추상화합니다.

## 사용 시점

- URL 기반 딥링크/유니버설 링크 지원이 필요할 때
- 인증 상태에 따른 리다이렉트 로직이 필요할 때
- BottomNavigationBar 등 중첩 네비게이션(ShellRoute)이 필요할 때
- 웹 플랫폼 지원으로 브라우저 URL 동기화가 필요할 때
- 경로 파라미터, 쿼리 파라미터를 체계적으로 관리해야 할 때

## 사용하지 말아야 할 때

- 풀스크린 다이얼로그, 바텀시트 등 오버레이 UI → `showDialog`, `showModalBottomSheet` 사용

## 필수 구현

### 라우터 설정

```dart
// app_router.dart
import 'package:go_router/go_router.dart';

final goRouter = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true, // 개발 중에만 활성화
  navigatorKey: _rootNavigatorKey,
  redirect: _globalRedirect, // 인증 등 전역 리다이렉트
  routes: [
    // ShellRoute: 하단 탭 등 공유 레이아웃
    StatefulShellRoute.indexedStack(
      builder: (context, state, navigationShell) {
        return MainScaffold(navigationShell: navigationShell);
      },
      branches: [
        StatefulShellBranch(
          navigatorKey: _homeNavigatorKey,
          routes: [
            GoRoute(
              path: '/',
              name: RouteNames.home,
              builder: (context, state) => const HomeScreen(),
              routes: [
                GoRoute(
                  path: 'detail/:id', // 경로 파라미터
                  name: RouteNames.detail,
                  builder: (context, state) {
                    final id = state.pathParameters['id']!;
                    return DetailScreen(id: id);
                  },
                ),
              ],
            ),
          ],
        ),
        StatefulShellBranch(
          navigatorKey: _settingsNavigatorKey,
          routes: [
            GoRoute(
              path: '/settings',
              name: RouteNames.settings,
              builder: (context, state) => const SettingsScreen(),
            ),
          ],
        ),
      ],
    ),
    // ShellRoute 밖의 풀스크린 라우트
    GoRoute(
      path: '/login',
      name: RouteNames.login,
      parentNavigatorKey: _rootNavigatorKey, // 루트 네비게이터에서 표시
      builder: (context, state) => const LoginScreen(),
    ),
  ],
);
```

### 라우트 이름 상수 관리

```dart
// route_names.dart
abstract final class RouteNames {
  static const home = 'home';
  static const detail = 'detail';
  static const settings = 'settings';
  static const login = 'login';
}
```

### 전역 리다이렉트

```dart
String? _globalRedirect(BuildContext context, GoRouterState state) {
  final isLoggedIn = context.read<AuthCubit>().state.isLoggedIn;
  final isLoginRoute = state.matchedLocation == '/login';

  // 미인증 사용자가 보호된 페이지 접근 시 로그인으로 리다이렉트
  if (!isLoggedIn && !isLoginRoute) {
    return '/login?redirect=${state.matchedLocation}';
  }

  // 인증된 사용자가 로그인 페이지 접근 시 홈으로 리다이렉트
  if (isLoggedIn && isLoginRoute) {
    return state.uri.queryParameters['redirect'] ?? '/';
  }

  return null; // 리다이렉트 없음
}
```

### 네비게이션 실행

```dart
// 선언적 네비게이션 — URL 스택 교체
context.go('/detail/123');

// 이름 기반 네비게이션 (권장)
context.goNamed(
  RouteNames.detail,
  pathParameters: {'id': '123'},
  queryParameters: {'tab': 'reviews'},
);

// push — 현재 스택 위에 추가 (뒤로가기 가능)
context.pushNamed(RouteNames.detail, pathParameters: {'id': '123'});

// pop — 이전 화면으로 돌아가기
context.pop();

// pushReplacement — 현재 화면을 교체
context.pushReplacementNamed(RouteNames.home);
```

### MaterialApp 연결

```dart
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: goRouter, // GoRouter 인스턴스 전달
    );
  }
}
```

### 핵심 규칙

1. **라우트 이름은 상수로 관리** — 문자열 하드코딩 금지. `RouteNames` 클래스에 정의하여 오타 방지
2. **`go` vs `push` 구분** — `go`는 URL 스택을 교체(웹 URL 변경), `push`는 스택 위에 추가(뒤로가기용). 목적에 맞게 선택
3. **리다이렉트는 순수 함수로 유지** — 비동기 호출, 사이드이펙트 없이 상태만 읽어서 경로 반환
4. **`parentNavigatorKey`로 풀스크린 제어** — ShellRoute 내부에서 탭 바 없이 전체 화면을 표시할 때 루트 네비게이터 키 지정
5. **경로 파라미터는 필수값, 쿼리 파라미터는 선택값** — `/:id`는 필수, `?tab=reviews`는 선택

## 아키텍처 / 구조

```
lib/
├── app/
│   ├── router/
│   │   ├── app_router.dart          # GoRouter 인스턴스 및 라우트 정의
│   │   ├── route_names.dart         # 라우트 이름 상수
│   │   └── guards/
│   │       └── auth_guard.dart      # 리다이렉트 로직
│   └── app.dart                     # MaterialApp.router
├── feature/
│   ├── home/
│   │   └── presentation/
│   │       └── view/
│   │           └── home_screen.dart
│   └── auth/
│       └── presentation/
│           └── view/
│               └── login_screen.dart
└── shared/
    └── widget/
        └── main_scaffold.dart       # ShellRoute용 공유 Scaffold
```

## 안티패턴

### 1. 문자열 경로 하드코딩

**잘못된 코드:**

```dart
// ❌ 경로 문자열이 여러 곳에 흩어져 오타 발생 시 런타임 에러
context.go('/home/detail/123');
context.go('/Home/Detail/123'); // 오타 — 404
```

**왜 잘못되었는가:** 경로 변경 시 모든 호출부를 수동 수정해야 하며, 오타를 컴파일 타임에 잡을 수 없습니다.

**올바른 방법:**

```dart
// ✅ 이름 기반 네비게이션 — 경로 구조 변경에 안전
context.goNamed(
  RouteNames.detail,
  pathParameters: {'id': '123'},
);
```

### 2. 리다이렉트에서 비동기 작업 수행

**잘못된 코드:**

```dart
// ❌ redirect는 동기 함수 — Future 반환 불가, 토큰 갱신 같은 비동기 로직 금지
String? _redirect(BuildContext context, GoRouterState state) {
  // 컴파일 에러는 아니지만 의도대로 동작하지 않음
  final token = await refreshToken(); // ❌ await 사용 불가
  if (token == null) return '/login';
  return null;
}
```

**왜 잘못되었는가:** `redirect`는 동기 함수이므로 비동기 작업을 넣으면 예상치 못한 동작이 발생합니다.

**올바른 방법:**

```dart
// ✅ 인증 상태는 상태 관리 도구에서 미리 갱신, redirect는 상태만 읽기
String? _redirect(BuildContext context, GoRouterState state) {
  final authState = context.read<AuthCubit>().state;
  if (!authState.isAuthenticated) return '/login';
  return null;
}
```

### 3. ShellRoute 내부 라우트에서 풀스크린 표시 누락

**잘못된 코드:**

```dart
// ❌ ShellRoute 내부에 정의 — 하단 탭 바가 그대로 표시됨
StatefulShellBranch(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
      routes: [
        GoRoute(
          path: 'photo-viewer/:id',
          builder: (context, state) => const PhotoViewerScreen(), // 탭 바 보임
        ),
      ],
    ),
  ],
),
```

**왜 잘못되었는가:** 몰입형 화면에서 하단 탭이 노출되어 UX가 깨집니다.

**올바른 방법:**

```dart
// ✅ parentNavigatorKey로 루트 네비게이터에서 표시
GoRoute(
  path: 'photo-viewer/:id',
  parentNavigatorKey: _rootNavigatorKey, // 루트에서 풀스크린
  builder: (context, state) => const PhotoViewerScreen(),
),
```

### 4. GoRouter를 Widget 내부에서 생성

**잘못된 코드:**

```dart
// ❌ Widget rebuild마다 GoRouter가 재생성 — 네비게이션 상태 초기화
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final router = GoRouter(routes: [...]); // 매번 새로 생성
    return MaterialApp.router(routerConfig: router);
  }
}
```

**왜 잘못되었는가:** rebuild마다 GoRouter가 재생성되면 네비게이션 스택이 초기화됩니다.

**올바른 방법:**

```dart
// ✅ 최상위에서 한 번만 생성
final goRouter = GoRouter(routes: [...]);

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(routerConfig: goRouter);
  }
}
```

### 5. go와 push 혼용

**잘못된 코드:**

```dart
// ❌ 리스트 → 상세로 이동할 때 go 사용
onTap: () => context.go('/detail/${item.id}'),
// 뒤로가기 시 리스트가 아닌 홈('/')으로 이동
```

**왜 잘못되었는가:** `go`는 URL 스택을 교체하므로 뒤로가기 시 기대한 화면이 아닌 다른 화면으로 돌아갑니다.

**올바른 방법:**

```dart
// ✅ 스택에 추가하여 뒤로가기 유지
onTap: () => context.pushNamed(
  RouteNames.detail,
  pathParameters: {'id': item.id},
),
```

## 의사결정 기준

### 네비게이션 방식 선택

| 기준      | `context.go`        | `context.push`     |
| --------- | ------------------- | ------------------ |
| URL 스택  | 교체                | 추가               |
| 뒤로가기  | 부모 경로로 이동    | 이전 화면으로 이동 |
| 웹 URL    | 변경됨              | 변경됨             |
| 주요 용도 | 탭 전환, 리다이렉트 | 상세 화면, 모달    |

### 라우트 타입 선택

| 기준                      | GoRoute | ShellRoute         | StatefulShellRoute |
| ------------------------- | ------- | ------------------ | ------------------ |
| 독립 화면                 | ✅      | ❌                 | ❌                 |
| 공유 레이아웃 (AppBar 등) | ❌      | ✅                 | ✅                 |
| 탭별 네비게이션 스택 유지 | ❌      | ❌                 | ✅                 |
| BottomNavigationBar       | ❌      | 가능하나 상태 소실 | ✅ 권장            |

**의사결정 규칙:** 독립 화면이면 GoRoute. 공유 레이아웃이 필요하고 탭 상태 유지가 불필요하면 ShellRoute. BottomNavigationBar처럼 탭별 스택 유지가 필요하면 StatefulShellRoute.

## 테스트 전략

### 라우트 설정 테스트

```dart
testWidgets('홈에서 상세로 네비게이션', (tester) async {
  final router = GoRouter(
    initialLocation: '/',
    routes: [
      GoRoute(
        path: '/',
        builder: (context, state) => const HomeScreen(),
        routes: [
          GoRoute(
            path: 'detail/:id',
            builder: (context, state) {
              final id = state.pathParameters['id']!;
              return DetailScreen(id: id);
            },
          ),
        ],
      ),
    ],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );

  router.go('/detail/42');
  await tester.pumpAndSettle();

  expect(find.byType(DetailScreen), findsOneWidget);
});
```

### 리다이렉트 테스트

```dart
testWidgets('미인증 사용자는 로그인으로 리다이렉트', (tester) async {
  final router = GoRouter(
    initialLocation: '/settings',
    redirect: (context, state) {
      final isLoggedIn = false; // 테스트용 미인증 상태
      if (!isLoggedIn && state.matchedLocation != '/login') {
        return '/login';
      }
      return null;
    },
    routes: [
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: '/settings',
        builder: (context, state) => const SettingsScreen(),
      ),
    ],
  );

  await tester.pumpWidget(
    MaterialApp.router(routerConfig: router),
  );
  await tester.pumpAndSettle();

  expect(find.byType(LoginScreen), findsOneWidget);
  expect(find.byType(SettingsScreen), findsNothing);
});
```

### Mock 해야 하는 것

- 인증 상태 (`AuthCubit`, `AuthRepository`) — 리다이렉트 분기 테스트를 위해

### Mock 하지 말아야 하는 것

- `GoRouter` 자체 — 실제 인스턴스로 라우팅 동작 검증
- `GoRouterState` — 실제 네비게이션으로 생성되는 state 사용

## AI가 자주 하는 실수

### 1. Navigator.push를 GoRouter 프로젝트에서 사용

**AI가 생성하는 코드:**

```dart
// ❌ GoRouter 프로젝트에서 Navigator 직접 사용
onTap: () {
  Navigator.of(context).push(
    MaterialPageRoute(
      builder: (context) => DetailScreen(id: item.id),
    ),
  );
}
```

**무엇이 잘못되었는가:** GoRouter의 URL 동기화, 리다이렉트, 딥링크가 모두 무시되어 뒤로가기/앞으로가기 동작이 깨집니다.

**올바른 접근법:**

```dart
// ✅ GoRouter의 context extension 사용
onTap: () => context.pushNamed(
  RouteNames.detail,
  pathParameters: {'id': item.id},
),
```

### 2. GoRoute의 builder에서 파라미터를 Widget 생성자로 전달하지 않음

**AI가 생성하는 코드:**

```dart
// ❌ state에서 파라미터를 꺼내지 않고 빈 생성자 호출
GoRoute(
  path: '/user/:userId/post/:postId',
  builder: (context, state) => const PostScreen(), // 파라미터 무시
),
```

**무엇이 잘못되었는가:** 경로 파라미터가 화면에 전달되지 않아 딥링크로 직접 접근 시 빈 화면이 표시됩니다.

**올바른 접근법:**

```dart
// ✅ pathParameters에서 추출하여 명시적으로 전달
GoRoute(
  path: '/user/:userId/post/:postId',
  builder: (context, state) {
    final userId = state.pathParameters['userId']!;
    final postId = state.pathParameters['postId']!;
    return PostScreen(userId: userId, postId: postId);
  },
),
```

### 3. StatefulShellRoute 대신 ShellRoute 사용하여 탭 상태 소실

**AI가 생성하는 코드:**

```dart
// ❌ ShellRoute로 BottomNavigationBar 구현 — 탭 전환 시 스크롤/상태 초기화
ShellRoute(
  builder: (context, state, child) {
    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        onTap: (index) => context.go(['/home', '/search', '/profile'][index]),
        items: const [...],
      ),
    );
  },
  routes: [...],
),
```

**무엇이 잘못되었는가:** `ShellRoute`는 탭 전환 시 이전 탭의 Widget 트리를 파괴하여 스크롤 위치 등 상태가 초기화됩니다.

**올바른 접근법:**

```dart
// ✅ StatefulShellRoute.indexedStack — 탭별 상태 보존
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) {
    return Scaffold(
      body: navigationShell, // IndexedStack으로 탭 상태 보존
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: navigationShell.currentIndex,
        onTap: (index) => navigationShell.goBranch(index),
        items: const [/* ... */],
      ),
    );
  },
  branches: [/* StatefulShellBranch per tab */],
),
```

## 참고 자료

- [go_router 패키지](https://pub.dev/packages/go_router) — 공식 API 문서 및 마이그레이션 가이드
- [Flutter 공식 Navigation 가이드](https://docs.flutter.dev/ui/navigation) — Flutter 네비게이션 개념 및 go_router 통합
- [go_router GitHub 예제](https://github.com/flutter/packages/tree/main/packages/go_router/example) — 공식 예제 코드 (ShellRoute, 리다이렉트 등)
