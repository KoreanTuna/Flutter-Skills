# Pagination 상태 관리

## 개요

BLoC에서 페이지네이션(초기 로드, 추가 로드, 새로고침)을 안전하게 관리하는 상태 머신 패턴입니다. `PagedState<T>` 제네릭 클래스로 초기 fetch와 load-more의 상태를 분리하고, 에러 발생 시 기존 데이터를 유지하면서 사용자에게 적절한 피드백을 제공합니다.

## 사용 시점

- 리스트 데이터를 페이지 단위로 불러와야 할 때 (무한 스크롤, 더보기 버튼)
- pull-to-refresh와 load-more를 동시에 지원해야 할 때
- 필터/정렬 변경 시 페이지네이션을 리셋해야 할 때
- 초기 로딩 에러와 추가 로딩 에러를 다르게 처리해야 할 때

## 사용하지 말아야 할 때

- 전체 데이터를 한 번에 가져오는 경우 → 단순 `AsyncStatus` + `List<T>` 사용
- 커서 기반 페이지네이션에서 서버가 다음 페이지 URL을 제공하는 경우 → `PagedState`를 커서 기반으로 확장
- 페이지네이션 없이 스트림으로 데이터가 실시간 유입되는 경우 → [BLoC Event Transformer](bloc-event-transformer.md) throttle 패턴 사용

## 필수 구현

### AsyncStatus 열거형

```dart
// core/util/async_status.dart
enum AsyncStatus {
  initial,
  loading,
  success,
  error,
}
```

### PagedState 제네릭 클래스

```dart
// core/util/paged_state.dart
import 'package:equatable/equatable.dart';

const _sentinel = Object();

class PagedState<T> extends Equatable {
  final AsyncStatus fetchStatus;      // 초기 로드 / 새로고침 상태
  final AsyncStatus fetchMoreStatus;  // 추가 로드 상태 (분리!)
  final List<T> items;
  final bool hasMore;
  final int currentPage;
  final String? errorMessage;

  const PagedState({
    required this.fetchStatus,
    required this.fetchMoreStatus,
    required this.items,
    required this.hasMore,
    required this.currentPage,
    required this.errorMessage,
  });

  factory PagedState.initial() {
    return PagedState<T>(
      fetchStatus: AsyncStatus.initial,
      fetchMoreStatus: AsyncStatus.initial,
      items: const [],
      hasMore: true,
      currentPage: 0,
      errorMessage: null,
    );
  }

  // sentinel 패턴으로 nullable 필드의 명시적 null 할당 지원
  PagedState<T> copyWith({
    AsyncStatus? fetchStatus,
    AsyncStatus? fetchMoreStatus,
    List<T>? items,
    bool? hasMore,
    int? currentPage,
    Object? errorMessage = _sentinel,
  }) {
    return PagedState<T>(
      fetchStatus: fetchStatus ?? this.fetchStatus,
      fetchMoreStatus: fetchMoreStatus ?? this.fetchMoreStatus,
      items: items ?? this.items,
      hasMore: hasMore ?? this.hasMore,
      currentPage: currentPage ?? this.currentPage,
      errorMessage: errorMessage == _sentinel
          ? this.errorMessage
          : errorMessage as String?,
    );
  }

  @override
  List<Object?> get props => [
    fetchStatus,
    fetchMoreStatus,
    items,
    hasMore,
    currentPage,
    errorMessage,
  ];
}
```

### BLoC Event 정의

```dart
part of 'post_list_bloc.dart';

sealed class PostListEvent extends Equatable {
  const PostListEvent();
  @override
  List<Object?> get props => [];
}

/// 초기 로드
class PostListRequested extends PostListEvent {
  const PostListRequested();
}

/// pull-to-refresh (Completer로 refresh indicator 종료 보장)
class PostListRefreshed extends PostListEvent {
  final Completer<void>? completer;
  const PostListRefreshed({this.completer});
  @override
  List<Object?> get props => [completer];
}

/// 추가 로드 (무한 스크롤)
class PostListLoadMoreRequested extends PostListEvent {
  const PostListLoadMoreRequested();
}

/// 필터 변경 — 페이지네이션 리셋
class PostListFilterChanged extends PostListEvent {
  final String? keyword;
  final int? categoryId;
  const PostListFilterChanged({this.keyword, this.categoryId});
  @override
  List<Object?> get props => [keyword, categoryId];
}

/// 정렬 변경 — 페이지네이션 리셋
class PostListSortChanged extends PostListEvent {
  final PostSort sort;
  const PostListSortChanged({required this.sort});
  @override
  List<Object?> get props => [sort];
}
```

### BLoC State 정의

```dart
part of 'post_list_bloc.dart';

class PostListState extends Equatable {
  final PagedState<Post> pagedState;
  final String? keyword;
  final int? categoryId;
  final PostSort sort;

  const PostListState({
    required this.pagedState,
    this.keyword,
    this.categoryId,
    this.sort = PostSort.latest,
  });

  factory PostListState.initial() => PostListState(
    pagedState: PagedState<Post>.initial(),
  );

  PostListState copyWith({
    PagedState<Post>? pagedState,
    Object? keyword = _sentinel,
    Object? categoryId = _sentinel,
    PostSort? sort,
  }) {
    return PostListState(
      pagedState: pagedState ?? this.pagedState,
      keyword: keyword == _sentinel ? this.keyword : keyword as String?,
      categoryId: categoryId == _sentinel ? this.categoryId : categoryId as int?,
      sort: sort ?? this.sort,
    );
  }

  @override
  List<Object?> get props => [pagedState, keyword, categoryId, sort];
}
```

### BLoC Handler 구현

```dart
@injectable
class PostListBloc extends Bloc<PostListEvent, PostListState> {
  final GetPostListUseCase _getPostListUseCase;
  static const _pageSize = 20;

  PostListBloc(this._getPostListUseCase)
      : super(PostListState.initial()) {
    on<PostListRequested>(
      _onRequested,
      transformer: droppableTransformer(),
    );
    on<PostListRefreshed>(
      _onRefreshed,
      transformer: droppableTransformer(),
    );
    on<PostListLoadMoreRequested>(
      _onLoadMoreRequested,
      transformer: droppableTransformer(),
    );
    on<PostListFilterChanged>(
      _onFilterChanged,
      transformer: droppableTransformer(),
    );
    on<PostListSortChanged>(
      _onSortChanged,
      transformer: droppableTransformer(),
    );
  }

  /// 초기 로드 — 페이지 1부터 시작
  Future<void> _onRequested(
    PostListRequested event,
    Emitter<PostListState> emit,
  ) async {
    emit(state.copyWith(
      pagedState: state.pagedState.copyWith(
        fetchStatus: AsyncStatus.loading,
        errorMessage: null, // sentinel 아닌 null로 초기화
      ),
    ));

    final result = await _getPostListUseCase.call(
      page: 1,
      pageSize: _pageSize,
      keyword: state.keyword,
      categoryId: state.categoryId,
      sort: state.sort,
    );

    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchStatus: AsyncStatus.success,
            items: value,
            currentPage: 1,
            hasMore: value.length >= _pageSize,
          ),
        ));
      case Error(:final error):
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchStatus: AsyncStatus.error,
            errorMessage: error.message,
          ),
        ));
    }
  }

  /// 추가 로드 — 기존 데이터 유지 + 다음 페이지 append
  Future<void> _onLoadMoreRequested(
    PostListLoadMoreRequested event,
    Emitter<PostListState> emit,
  ) async {
    // Guard: 더 이상 데이터 없거나 이미 로딩 중이면 무시
    if (!state.pagedState.hasMore) return;
    if (state.pagedState.fetchMoreStatus == AsyncStatus.loading) return;

    final nextPage = state.pagedState.currentPage + 1;

    emit(state.copyWith(
      pagedState: state.pagedState.copyWith(
        fetchMoreStatus: AsyncStatus.loading,
      ),
    ));

    final result = await _getPostListUseCase.call(
      page: nextPage,
      pageSize: _pageSize,
      keyword: state.keyword,
      categoryId: state.categoryId,
      sort: state.sort,
    );

    switch (result) {
      case Ok(:final value):
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchMoreStatus: AsyncStatus.success,
            items: [...state.pagedState.items, ...value], // 기존 + 신규 append
            currentPage: nextPage,
            hasMore: value.length >= _pageSize,
          ),
        ));
      case Error(:final error):
        // 추가 로드 실패 — 기존 items 유지, fetchMoreStatus만 error
        emit(state.copyWith(
          pagedState: state.pagedState.copyWith(
            fetchMoreStatus: AsyncStatus.error,
            errorMessage: error.message,
          ),
        ));
    }
  }

  /// pull-to-refresh — 페이지 1로 리셋, Completer로 indicator 종료
  Future<void> _onRefreshed(
    PostListRefreshed event,
    Emitter<PostListState> emit,
  ) async {
    try {
      final result = await _getPostListUseCase.call(
        page: 1,
        pageSize: _pageSize,
        keyword: state.keyword,
        categoryId: state.categoryId,
        sort: state.sort,
      );

      switch (result) {
        case Ok(:final value):
          emit(state.copyWith(
            pagedState: state.pagedState.copyWith(
              fetchStatus: AsyncStatus.success,
              items: value, // 전체 교체 (append 아님)
              currentPage: 1,
              hasMore: value.length >= _pageSize,
              errorMessage: null,
            ),
          ));
        case Error(:final error):
          emit(state.copyWith(
            pagedState: state.pagedState.copyWith(
              errorMessage: error.message,
            ),
          ));
      }
    } finally {
      event.completer?.complete(); // refresh indicator 종료 보장
    }
  }

  /// 필터 변경 — 페이지네이션 리셋 후 재요청
  Future<void> _onFilterChanged(
    PostListFilterChanged event,
    Emitter<PostListState> emit,
  ) async {
    emit(state.copyWith(
      keyword: event.keyword,
      categoryId: event.categoryId,
      pagedState: PagedState<Post>.initial(), // 완전 리셋
    ));
    add(const PostListRequested()); // 새 필터로 재요청
  }

  // _onSortChanged도 동일 패턴: sort 필드 업데이트 → PagedState.initial() 리셋 → PostListRequested() 재요청
}
```

### 핵심 규칙

1. **`fetchStatus`와 `fetchMoreStatus`를 반드시 분리** — 초기 로드 실패 시 전체 에러 화면, 추가 로드 실패 시 하단 토스트
2. **`hasMore` 가드를 항상 확인** — load-more 전에 `hasMore == false`이면 요청하지 않음
3. **refresh 시 `items`를 전체 교체, load-more 시 append** — `[...existing, ...newItems]`
4. **필터/정렬 변경 시 `PagedState.initial()`로 완전 리셋** — 이전 페이지 데이터와 새 필터 데이터가 섞이는 것을 방지
5. **Completer는 `finally`에서 완료** — refresh indicator가 무한 로딩 상태에 빠지는 것을 방지
6. **`copyWith`에서 nullable 필드는 sentinel 패턴 사용** — `errorMessage: null` 할당을 위해

## 아키텍처 / 구조

```
lib/
├── core/
│   └── util/
│       ├── async_status.dart           # AsyncStatus enum
│       └── paged_state.dart            # PagedState<T> 제네릭 클래스
│
└── feature/{domain}/presentation/
    └── bloc/{list_bloc}/
        ├── {list}_bloc.dart            # 페이지네이션 로직
        ├── {list}_event.dart           # Requested, Refreshed, LoadMore, Filter, Sort
        └── {list}_state.dart           # PagedState<T>를 포함하는 State
```

## 안티패턴

### 1. fetchStatus 하나로 초기 로드와 추가 로드를 관리

**잘못된 코드:**

```dart
class PostListState extends Equatable {
  final AsyncStatus status; // ❌ 초기 로드와 추가 로드를 구분할 수 없음
  final List<Post> posts;
  final bool isLoadingMore; // ❌ 별도 bool로 관리 — status와 불일치 가능
  // ...
}
```

**왜 잘못되었는가:** 초기 로드와 추가 로드를 구분할 수 없어 UI 분기가 불가능하고 `isLoadingMore`와 `status` 동기화 문제가 발생합니다.

**올바른 방법:**

```dart
class PostListState extends Equatable {
  final PagedState<Post> pagedState; // fetchStatus + fetchMoreStatus 분리
  // ...
}

// UI에서 분기
if (state.pagedState.fetchStatus == AsyncStatus.loading) {
  return const FullScreenLoader(); // 초기 로드 — 전체 스피너
}
if (state.pagedState.fetchMoreStatus == AsyncStatus.loading) {
  // 하단 로딩 인디케이터
}
if (state.pagedState.fetchMoreStatus == AsyncStatus.error) {
  // 하단 재시도 버튼 (기존 리스트 유지)
}
```

### 2. load-more에서 기존 데이터를 교체

**잘못된 코드:**

```dart
Future<void> _onLoadMore(LoadMore event, Emitter<State> emit) async {
  final result = await _useCase.call(page: nextPage);
  switch (result) {
    case Ok(:final value):
      emit(state.copyWith(items: value)); // ❌ 기존 데이터 소실
  }
}
```

**왜 잘못되었는가:** 다음 페이지 로드 시 이전 페이지 데이터가 소실됩니다.

**올바른 방법:**

```dart
case Ok(:final value):
  emit(state.copyWith(
    items: [...state.pagedState.items, ...value], // 기존 + 신규 append
  ));
```

### 3. 필터 변경 시 페이지네이션 리셋 누락

**잘못된 코드:**

```dart
Future<void> _onFilterChanged(FilterChanged event, Emitter<State> emit) async {
  emit(state.copyWith(keyword: event.keyword));
  // ❌ currentPage, hasMore, items가 이전 필터의 값 유지
  add(const FetchRequested());
}
```

**왜 잘못되었는가:** 이전 필터의 페이지/데이터 상태가 새 필터에 그대로 남아 데이터가 섞입니다.

**올바른 방법:**

```dart
Future<void> _onFilterChanged(FilterChanged event, Emitter<State> emit) async {
  emit(state.copyWith(
    keyword: event.keyword,
    pagedState: PagedState<Post>.initial(), // 완전 리셋
  ));
  add(const PostListRequested()); // 1페이지부터 재요청
}
```

### 4. Completer 미사용으로 refresh indicator 무한 로딩

**잘못된 코드:**

```dart
RefreshIndicator(
  onRefresh: () async {
    context.read<PostListBloc>().add(const PostListRefreshed());
    // ❌ bloc.add()는 fire-and-forget — indicator가 즉시 사라짐
  },
)
```

**왜 잘못되었는가:** `bloc.add()`는 Future를 반환하지 않아 RefreshIndicator가 완료 시점을 알 수 없습니다.

**올바른 방법:** Completer를 Event에 담아 전달하고 Handler의 `finally`에서 complete — 필수 구현 섹션의 `_onRefreshed` 참조.

## 의사결정 기준

### 페이지 기반 vs 커서 기반

| 기준 | 페이지 기반 (offset) | 커서 기반 (cursor) |
|---|---|---|
| 서버 API | `?page=2&size=20` | `?cursor=abc123&size=20` |
| 데이터 변경 시 | 중복/누락 가능 | 안정적 |
| 총 페이지 수 | 계산 가능 | 불가 |
| 구현 복잡도 | 낮음 | 중간 |
| 적합한 상황 | 정적 데이터, 관리자 페이지 | 실시간 피드, SNS 타임라인 |

**의사결정 규칙:** 서버가 커서를 제공하면 커서 기반 사용. 아니면 페이지 기반으로 시작하고, 데이터 정합성 이슈가 발생하면 커서 기반으로 전환.

### hasMore 판단 기준

| 방법 | 로직 | 장점 | 단점 |
|---|---|---|---|
| 응답 개수 비교 | `value.length < pageSize` | 추가 API 없음 | 정확히 pageSize개일 때 1회 추가 요청 |
| 서버 totalCount | `items.length >= totalCount` | 정확함 | 서버에서 total 제공 필요 |
| 서버 hasNext 필드 | `response.hasNext` | 가장 정확 | 서버 API 변경 필요 |

**의사결정 규칙:** `value.length >= pageSize`이면 `hasMore: true`로 판단. 1회 추가 빈 요청은 허용.

## 테스트 전략

```dart
void main() {
  late PostListBloc bloc;
  late FakePostQueryRepository fakeRepository;

  setUp(() {
    fakeRepository = FakePostQueryRepository();
    bloc = PostListBloc(GetPostListUseCase(fakeRepository));
  });

  tearDown(() => bloc.close());

  blocTest<PostListBloc, PostListState>(
    '초기 로드 성공 시 items가 설정되고 hasMore가 올바르다',
    build: () {
      fakeRepository.postsToReturn = List.generate(20, (i) => Post(id: '$i'));
      return bloc;
    },
    act: (bloc) => bloc.add(const PostListRequested()),
    expect: () => [
      // loading
      predicate<PostListState>(
        (s) => s.pagedState.fetchStatus == AsyncStatus.loading,
      ),
      // success
      predicate<PostListState>(
        (s) =>
            s.pagedState.fetchStatus == AsyncStatus.success &&
            s.pagedState.items.length == 20 &&
            s.pagedState.currentPage == 1 &&
            s.pagedState.hasMore == true,
      ),
    ],
  );

  blocTest<PostListBloc, PostListState>(
    'load-more 시 기존 items에 append된다',
    seed: () => PostListState(
      pagedState: PagedState<Post>(
        fetchStatus: AsyncStatus.success,
        fetchMoreStatus: AsyncStatus.initial,
        items: List.generate(20, (i) => Post(id: '$i')),
        hasMore: true,
        currentPage: 1,
        errorMessage: null,
      ),
    ),
    build: () {
      fakeRepository.postsToReturn = List.generate(20, (i) => Post(id: '${i + 20}'));
      return bloc;
    },
    act: (bloc) => bloc.add(const PostListLoadMoreRequested()),
    expect: () => [
      // fetchMoreStatus: loading
      predicate<PostListState>(
        (s) => s.pagedState.fetchMoreStatus == AsyncStatus.loading,
      ),
      // fetchMoreStatus: success, items: 40개
      predicate<PostListState>(
        (s) =>
            s.pagedState.fetchMoreStatus == AsyncStatus.success &&
            s.pagedState.items.length == 40 &&
            s.pagedState.currentPage == 2,
      ),
    ],
  );

  // load-more 실패 테스트: 위 성공 테스트와 동일한 seed 사용,
  // fakeRepository.errorToThrow 설정 후 검증:
  // - fetchMoreStatus: loading → error
  // - items.length == 20 (기존 데이터 유지)
}
```

**Mock 대상:** UseCase (Repository 격리) | **Mock 금지:** PagedState (상태 전이 검증), Event Transformer (droppable 동작 포함 테스트)

## AI가 자주 하는 실수

### 1. 단일 status로 초기/추가 로드 관리

**AI가 생성하는 코드:**

```dart
class ListState extends Equatable {
  final AsyncStatus status;
  final List<Item> items;
  final bool hasMore;
}

// Handler
Future<void> _onLoadMore(LoadMore event, Emitter<ListState> emit) async {
  emit(state.copyWith(status: AsyncStatus.loading)); // ❌ 전체 화면이 로딩으로 전환
  final result = await _useCase.call(page: nextPage);
  // ...
}
```

**무엇이 잘못되었는가:** 추가 로드 시 전체 화면이 로딩 스피너로 전환되어 기존 리스트가 사라지는 UX 문제가 발생합니다.

**올바른 접근법:**

```dart
class ListState extends Equatable {
  final PagedState<Item> pagedState; // fetchStatus + fetchMoreStatus 분리
}

// 추가 로드 — fetchMoreStatus만 변경, fetchStatus(전체 화면)는 유지
emit(state.copyWith(
  pagedState: state.pagedState.copyWith(
    fetchMoreStatus: AsyncStatus.loading, // 하단 인디케이터만 표시
  ),
));
```

### 2. hasMore 가드 없이 무한 요청

**AI가 생성하는 코드:**

```dart
Future<void> _onLoadMore(LoadMore event, Emitter<State> emit) async {
  // ❌ hasMore 체크 없음 — 데이터 끝에 도달해도 계속 API 호출
  final nextPage = state.currentPage + 1;
  final result = await _useCase.call(page: nextPage);
  // ...
}
```

**무엇이 잘못되었는가:** 마지막 페이지 이후에도 무한히 빈 요청을 보내 불필요한 네트워크/서버 비용이 발생합니다.

**올바른 접근법:**

```dart
Future<void> _onLoadMore(LoadMore event, Emitter<State> emit) async {
  if (!state.pagedState.hasMore) return; // 더 이상 데이터 없으면 무시
  if (state.pagedState.fetchMoreStatus == AsyncStatus.loading) return; // 중복 방지
  // ...
}
```

## 참고 자료

- [flutter_bloc 패키지](https://pub.dev/packages/flutter_bloc) — BLoC 상태 관리
- [equatable 패키지](https://pub.dev/packages/equatable) — State 동등성 비교
- [bloc_concurrency 패키지](https://pub.dev/packages/bloc_concurrency) — droppable Transformer (중복 요청 방지)
