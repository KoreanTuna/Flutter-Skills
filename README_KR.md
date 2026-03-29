# Flutter-Skills

AI 바이브 코딩 어시스턴트를 위한 Flutter 개발 가이드

**[English](README.md)** | **한국어**

## 목적

AI 어시스턴트(및 개발자)가 Flutter 개발 시 올바른 구현 결정을 내릴 수 있도록 돕는 체계적이고 의견이 담긴 가이드입니다.

AI와 바이브 코딩을 할 때, AI는 종종 Flutter API를 잘못 사용하거나, 잘못된 패턴을 선택하거나, 발견하기 어려운 안티패턴을 생성합니다. 이 프로젝트는 **토픽별 가이드**를 통해 필수 패턴, 안티패턴, 의사결정 기준, 그리고 AI가 자주 하는 실수를 다룹니다 — AI가 처음부터 올바르게 코드를 작성할 수 있도록.

> 5년간의 프로덕션 Flutter 개발 경험을 바탕으로 작성되었습니다.

## 사용 방법

### AI 기반 개발 (바이브 코딩)

작업을 시작하기 **전에** AI 어시스턴트에게 관련 가이드를 읽도록 지시하세요:

```
"이 프로젝트에서 BLoC 코드를 생성하기 전에 `state-management/bloc-pattern.md`를 읽어줘."
```

또는 여러 가이드를 함께 로드할 수 있습니다:

```
"새 기능을 구현하기 전에 `architecture/clean-architecture.md`와 `state-management/bloc-pattern.md`를 읽고 프로젝트 구조를 이해해줘."
```

### 개발자용

아래 카테고리별로 토픽을 찾아보세요. 모든 가이드는 [`TEMPLATE.md`](TEMPLATE.md)에 정의된 일관된 템플릿을 따릅니다.

## 토픽

| 카테고리 | 가이드 |
|---|---|
| **상태 관리** | [BLoC 패턴](state-management/bloc-pattern.md), [BLoC Event Transformer](state-management/bloc-event-transformer.md), [Pagination 상태 관리](state-management/pagination-state.md), [Flutter Hook 로컬 상태 관리](state-management/local-state-management-by-flutter-hook.md) |
| **아키텍처** | [BLoC 레이어 아키텍처 적용](architecture/bloc-apply-layer-architecture.md), [CQRS 레이어 아키텍처](architecture/cqrs-layer-architecture.md), [데이터 모델링](architecture/data-modeling.md) |
| **내비게이션** | [GoRouter 네비게이션](navigation/go-router-navigation.md) |
| **의존성 주입** | [GetIt & Injectable](dependency-injection/get-it-injectable.md) |
| **에러 처리** | [Result 패턴](error-handling/result-pattern.md) |
| **성능 최적화** | [RepaintBoundary](performance/repaint-boundary.md) |
| **영속성** | [SharedPreferences](persistence/shared-preferences.md), [SecureStorage](persistence/secure-storage.md) |
| **네트워킹** | [Retry & Exponential Backoff](networking/retry-exponential-backoff.md) |

## 가이드 형식

모든 가이드는 [`TEMPLATE.md`](TEMPLATE.md)의 템플릿을 따르며 다음 섹션으로 구성됩니다:

| 섹션 | 목적 |
|---|---|
| **개요 (Overview)** | 이 토픽이 다루는 내용과 적용 시점 |
| **사용 시점 (When To Use)** | 이 접근법이 올바른 선택인 시나리오 |
| **사용하지 말아야 할 때 (When NOT To Use)** | 잘못된 시나리오와 대안 안내 |
| **필수 구현 (Required Implementation)** | 반드시 따라야 하는 코드 패턴 |
| **아키텍처 / 구조 (Architecture / Structure)** | 권장 파일/폴더 레이아웃 |
| **안티패턴 (Anti-Patterns)** | 피해야 할 것과 그 이유 |
| **의사결정 기준 (Decision Criteria)** | 여러 옵션이 있을 때 비교 테이블 |
| **테스트 전략 (Testing Strategy)** | 이 패턴을 사용하는 코드의 테스트 방법 |
| **AI가 자주 하는 실수 (Common AI Mistakes)** | AI 어시스턴트가 이 영역에서 자주 범하는 오류 |
| **참고 자료 (References)** | 공식 문서 및 핵심 리소스 |

## 프로젝트 구조

```
Flutter-Skills/
├── README.md
├── README_KR.md
├── TEMPLATE.md
├── CONTRIBUTING.md
│
├── state-management/          # 상태 관리
│   ├── bloc-pattern.md
│   ├── bloc-event-transformer.md
│   ├── pagination-state.md
│   └── local-state-management-by-flutter-hook.md
│
├── architecture/              # 아키텍처
│   ├── bloc-apply-layer-architecture.md
│   ├── cqrs-layer-architecture.md
│   └── data-modeling.md
│
├── navigation/                # 내비게이션
│   └── go-router-navigation.md
│
├── dependency-injection/      # 의존성 주입
│   └── get-it-injectable.md
│
├── error-handling/            # 에러 처리
│   └── result-pattern.md
│
├── persistence/               # 영속성
│   ├── shared-preferences.md
│   └── secure-storage.md
│
├── performance/               # 성능 최적화
│   └── repaint-boundary.md
│
└── networking/                # 네트워킹
    └── retry-exponential-backoff.md
```

## 기여하기

가이드를 추가하거나 개선하려면 [`CONTRIBUTING.md`](CONTRIBUTING.md)를 참고하세요.

## 라이선스

이 프로젝트는 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 라이선스 하에 배포됩니다 — 출처를 밝히면 자유롭게 공유하고 수정할 수 있습니다.
