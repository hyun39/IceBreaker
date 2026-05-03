# Git 레포지토리 구조 제안 요약

> 원본 스펙: `specs/edit/enterprise/02.04_gov_repository_structure.md`  
> 작성일: 2026-05-03

---

## 전체 레포 구조

```
github.com/corp/
│
├── infra/              ← GitOps 중앙 (ArgoCD + Kyverno + Helm)
├── platform-sdk/       ← @corp/sdk-python, @corp/sdk-java (모노레포)
├── design-system/      ← @corp/design-system + Design Tokens (모노레포)
├── data-platform/      ← 전사 Airflow DAG + dbt + Data Contract
├── specs/              ← enterprise/ 스펙 + 서비스 스펙 허브
│
└── {service}/          ← 서비스별 독립 레포 (멀티레포)
    ├── specs/          ← 이 서비스의 비즈니스·기술 스펙 + ADR
    ├── apps/api/       ← FastAPI / Spring Boot
    ├── apps/frontend/  ← React SPA
    ├── airflow/dags/   ← 서비스 전용 DAG
    ├── contracts/      ← Pact + Data Contract
    └── .github/        ← CI(lint·test·build) + prototype.yml + release.yml
```

---

## 레포 유형별 핵심 의사결정

| 레포 | 멀티레포 vs 모노레포 | 이유 |
|------|-------------------|------|
| 서비스(`{service}`) | **멀티레포** | 팀 독립 배포, 다른 릴리스 주기 |
| 공유 라이브러리 | **모노레포** (Turborepo / uv workspace) | 내부 패키지 간 참조 편의 |
| 데이터 플랫폼 | **모노레포** (단일 레포) | DAG·dbt·Contract 상호 참조 |
| GitOps 인프라 | **단일 레포** | ArgoCD 단일 감시 포인트 |

---

## 단방향 의존 원칙

```
허용 ✅
  {service} → platform-sdk     (서비스가 SDK 소비)
  {service} → design-system    (프론트엔드만)
  infra     → 없음             (infra는 소비만 됨)

금지 ❌
  platform-sdk → {service}     (순환 의존)
  {service}    → {service}     (직접 import 금지)
                               → API 또는 Kafka 이벤트로만 통신
```

---

## 서비스 레포 내부 구조 (Golden Path 표준)

```
{service}/
├── specs/
│   ├── business_spec.md         ← 비즈니스 스펙
│   ├── api_contract.md          ← API 계약
│   ├── software_architecture.md
│   └── adr/
│       ├── README.md            ← ADR 인덱스
│       └── ADR-0001-xxx.md
│
├── apps/
│   ├── api/                     ← FastAPI / Spring Boot
│   │   ├── app/
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── contract/        ← Pact Consumer 테스트
│   │   └── Dockerfile
│   │
│   └── frontend/                ← React SPA
│       ├── src/
│       │   ├── pages/
│       │   ├── components/
│       │   ├── mocks/           ← MSW 핸들러
│       │   └── store/
│       └── Dockerfile
│
├── airflow/dags/                ← 서비스 전용 DAG
├── contracts/                   ← Pact + Data Contract
├── docs/                        ← TechDocs (Backstage 자동 발행)
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── prototype.yml        ← 스펙 변경 시 React 목업 자동 생성
│   │   └── release.yml          ← semantic-release → image tag → infra PR
│   └── CODEOWNERS
└── catalog-info.yaml            ← Backstage 등록
```

---

## infra/ 레포 구조 (GitOps 중앙)

```
infra/
├── apps/                        ← ArgoCD Application 정의
│   ├── dev/
│   │   ├── _root-app.yaml       ← App of Apps
│   │   └── {service}.yaml
│   ├── staging/
│   └── prod/
│
├── helm/
│   ├── base-service/            ← 모든 서비스가 상속하는 Base Chart
│   └── {service}/
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-staging.yaml
│       └── values-prod.yaml
│
└── policies/                    ← Kyverno 정책
    ├── require-signed-image.yaml
    └── disallow-root.yaml
```

---

## 배포 흐름 요약

```
개발자 → {service} 레포 PR → main 머지
    │
    ├── CI: lint → test → build → image push (git-sha)
    └── release.yml: semantic-release → image tag → infra 레포 PR 자동 생성
                                                          │
                                                    ArgoCD 감지
                                                          │
                                                    K8s 자동 배포
```

---

## 관련 Enterprise 스펙 파일

| 스펙 | 내용 |
|------|------|
| `02.02_gov_branch_strategy.md` | 브랜치 보호·CODEOWNERS |
| `02.03_gov_version_release_management.md` | semantic-release·SemVer |
| `02.04_gov_repository_structure.md` | 전체 레포 구조 상세 (원본) |
| `04.01_std_platform_engineering.md` | Backstage Golden Path 템플릿 |
| `04.03_std_gitops.md` | infra 레포 ArgoCD 구성 |
| `05.01_std_application_standards.md` | 서비스 내부 코드 구조 |
