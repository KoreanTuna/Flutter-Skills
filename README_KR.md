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
| **상태 관리** | BLoC 패턴, Cubit 패턴, BLoC vs Cubit, Provider, Riverpod, State Restoration |
| **아키텍처** | 클린 아키텍처, Feature-first vs Layer-first, Repository 패턴, 도메인 모델링 |
| **내비게이션** | GoRouter, Navigator 2.0, 딥링킹, BLoC과 내비게이션 |
| **의존성 주입** | GetIt, Injectable, Service Locator vs DI |
| **테스팅** | 단위 테스트, 위젯 테스트, 통합 테스트, 테스트 가능한 코드 패턴 |
| **네트워킹** | Dio 설정, Retrofit, API 에러 처리, 오프라인 우선 |
| **영속성** | SharedPreferences, Drift, Hive/Isar, Secure Storage |
| **성능** | 위젯 리빌드 최적화, 리스트 성능, 이미지 최적화, Isolate, DevTools 프로파일링 |
| **UI** | 테마 & 디자인 시스템, 반응형 디자인, 애니메이션, CustomPaint, 폼 & 유효성 검사 |
| **에러 처리** | 에러 처리 패턴, Crashlytics 통합, 글로벌 에러 바운더리 |
| **플랫폼** | Platform Channels, Pigeon, FFI, 플랫폼별 UI |
| **국제화** | ARB 현지화, Intl 모범 사례 |
| **Firebase** | 설정, 인증, Firestore 패턴, Cloud Functions |
| **CI/CD** | GitHub Actions, Fastlane, 코드 품질 |

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
├── architecture/              # 아키텍처
├── navigation/                # 내비게이션
├── dependency-injection/      # 의존성 주입
├── testing/                   # 테스팅
├── networking/                # 네트워킹
├── persistence/               # 영속성
├── performance/               # 성능
├── ui/                        # UI
├── error-handling/            # 에러 처리
├── platform/                  # 플랫폼
├── internationalization/      # 국제화
├── firebase/                  # Firebase
└── ci-cd/                     # CI/CD
```

## 기여하기

가이드를 추가하거나 개선하려면 [`CONTRIBUTING.md`](CONTRIBUTING.md)를 참고하세요.

## 라이선스

이 프로젝트는 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 라이선스 하에 배포됩니다 — 출처를 밝히면 자유롭게 공유하고 수정할 수 있습니다.
