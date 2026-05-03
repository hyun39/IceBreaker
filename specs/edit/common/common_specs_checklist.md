# Common Spec — 요소별 질문형 체크리스트 (행 분리)

요청하신 대로 **체크리스트 질문 1개 = 표 1개 row** 형태로 구성했습니다.

| 항목 | 체크리스트 | 선택후부 | 미고려시 예상 상황 | 설명 |
|------|------------|----------|--------------------|------|
| ADR 포맷 | ADR 번호와 상태(제안/수락/폐기/대체)를 명확히 기록했는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| ADR 포맷 | 배경에 제약사항(기술, 일정, 조직)을 구체적으로 적었는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| ADR 포맷 | 검토한 옵션별 장단점이 비교 가능한 수준으로 작성됐는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| ADR 포맷 | 선택한 옵션의 이유가 왜 중심으로 명시됐는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| ADR 포맷 | 수락된 ADR을 수정하지 않고 새 ADR로 대체했는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| Agentic Agent | 목표 작업에 맞는 에이전트 패턴(ReAct, Plan-Execute 등)을 선택했는가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| Agentic Agent | `max_iterations`와 timeout을 설정해 무한 실행을 방지했는가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| Agentic Agent | 파싱 오류 및 툴 실패 시 재시도/대체 흐름을 정의했는가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| Agentic Agent | 툴 description이 입력/출력/사용 시점을 충분히 설명하는가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| Agentic Agent | 트레이스/로그로 Thought-Action-Observation을 추적 가능한가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| API Gateway | 단일 진입점으로 TLS 종료와 라우팅을 일관되게 처리하는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| API Gateway | JWT 검증 책임 경계를 게이트웨이와 백엔드에 명확히 나눴는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| API Gateway | Rate limit 정책(분, 시간, 사용자 등급)을 수치로 정의했는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| API Gateway | 타임아웃, 서킷브레이커, 재시도 정책이 합의됐는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| API Gateway | 헬스체크/관리 경로가 외부 노출 정책과 분리되어 있는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| 인증·인가(Keycloak) | 프론트 채널에서 Auth Code + PKCE를 적용했는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| 인증·인가(Keycloak) | JWT 서명, 만료, issuer, audience 검증을 모두 수행하는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| 인증·인가(Keycloak) | Realm 역할과 Client 역할을 API 권한과 정확히 매핑했는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| 인증·인가(Keycloak) | JWKS 캐시 갱신 주기와 키 로테이션 대응을 정의했는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| 인증·인가(Keycloak) | 브라우저나 공개 저장소에 client secret이 노출되지 않는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| Backend (FastAPI) | 라우터, 서비스, 클라이언트, 스키마 계층이 분리되어 있는가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (FastAPI) | Pydantic 제약조건과 `response_model`이 실제 계약과 일치하는가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (FastAPI) | 외부 호출에 timeout, retry, 에러 매핑을 적용했는가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (FastAPI) | 의존성 주입(`Depends`)으로 테스트 대체가 가능한 구조인가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (FastAPI) | 설정값과 시크릿이 환경별로 안전하게 분리되는가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (Spring Boot) | Controller-Service-Client 책임이 명확히 분리되어 있는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| Backend (Spring Boot) | `@Valid` 및 Bean Validation으로 입력 검증이 수행되는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| Backend (Spring Boot) | 공통 예외 처리(`@RestControllerAdvice`)로 에러 응답이 표준화됐는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| Backend (Spring Boot) | WebClient/RestClient 설정(타임아웃, 풀, 재시도)이 공통화됐는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| Backend (Spring Boot) | OpenAPI 문서와 실제 엔드포인트가 동기화되어 있는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| 브랜치 전략 | `main`/`develop` 보호 규칙(직접 push 금지, 필수 리뷰)을 설정했는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 브랜치 전략 | `feature/fix/release/hotfix` 네이밍 규칙을 팀에 공유했는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 브랜치 전략 | 머지 전략(Squash/Rebase/Merge)과 커밋 메시지 기준을 정했는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 브랜치 전략 | 핫픽스 후 develop 백포트 절차를 문서화했는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 브랜치 전략 | 브랜치별 CI 필수 체크가 보호 규칙과 일치하는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 비즈니스 스펙 포맷 | 문서가 무엇을/왜 중심으로 작성되고 어떻게를 배제했는가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 비즈니스 스펙 포맷 | 핵심 가치와 성공 기준이 측정 가능한 문장으로 표현됐는가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 비즈니스 스펙 포맷 | 주요 사용자 시나리오에 전제조건과 완료 기준이 있는가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 비즈니스 스펙 포맷 | 이해관계자 역할과 승인 책임이 명확히 적혀 있는가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 비즈니스 스펙 포맷 | 상태(초안/검토중/승인/폐기)와 버전이 최신인가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 캐싱 전략 | 캐시 계층(CDN, Gateway, Redis, DB)의 책임을 분리했는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 캐싱 전략 | 키 네이밍 규칙에 서비스명/버전/엔티티가 포함되는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 캐싱 전략 | 데이터 유형별 TTL과 갱신 트리거가 명시되어 있는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 캐싱 전략 | Cache-Aside 미스 처리와 무효화 이벤트를 정의했는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 캐싱 전략 | 캐시 히트율과 stale 비율을 모니터링하는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 데이터 거버넌스 | 데이터 등급(L1~L4)과 PII 목록이 최신 상태인가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| 데이터 거버넌스 | 수집 목적, 보존 기간, 삭제 방식이 데이터별로 정의됐는가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| 데이터 거버넌스 | 로그/추적 데이터에 PII 마스킹이 적용되는가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| 데이터 거버넌스 | 접근 권한이 최소 권한 원칙에 맞게 설정됐는가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| 데이터 거버넌스 | 데이터 계보(출처→저장→가공→노출)를 추적할 수 있는가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| Database | PK, FK, ON DELETE 정책이 테이블별로 일관적인가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| Database | 정규화/비정규화 선택 사유가 문서와 주석에 남아 있는가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| Database | 쿼리 패턴 기준으로 인덱스를 설계하고 `EXPLAIN`으로 검증했는가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| Database | `TIMESTAMPTZ`, JSONB 등 타입 선택 기준이 합의됐는가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| Database | 만료 데이터(TTL) 삭제 배치가 운영에 적용됐는가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| 배포 전략 | 동일 `git-sha` 이미지가 dev→staging→prod로 승격되는가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| 배포 전략 | 환경별 승인자 수와 승인 조건이 명확한가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| 배포 전략 | E2E 실패 시 다음 환경 배포를 자동 차단하는가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| 배포 전략 | Canary 단계별 기준(에러율, 지연시간, 대기시간)이 정의됐는가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| 배포 전략 | 롤백 명령과 담당자 연락체계를 사전에 점검했는가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| Frontend | 페이지별 특성에 맞게 CSR, SSR, SSG, Hybrid를 선택했는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| Frontend | 로딩, 빈 상태, 오류, 재시도 UI가 정의되어 있는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| Frontend | 요청 취소와 중복 요청 방지를 구현했는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| Frontend | 상태 관리 범위(로컬/전역)와 도구 선택 근거가 있는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| Frontend | 성능 지표(LCP, CLS 등)와 접근성 기준을 점검하는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| 인프라·CI/CD | `lint→test→build→image` 단계가 자동화되고 실패 시 차단되는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 인프라·CI/CD | 컨테이너가 멀티스테이지와 non-root 원칙을 따르는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 인프라·CI/CD | Readiness/Liveness probe, HPA, 리소스 제한을 설정했는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 인프라·CI/CD | 환경별 ConfigMap/Secret과 Helm 값 분리가 되어 있는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 인프라·CI/CD | 취약점 스캔과 이미지 서명이 배포 전 수행되는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 메시징·EDA | 브로커 선택 기준(처리량, 순서, 보존, 운영성)을 정의했는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 메시징·EDA | 이벤트 스키마에 `event_id`, `version`, `correlation_id`가 있는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 메시징·EDA | 멱등성 키와 중복 소비 방지 로직이 구현되어 있는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 메시징·EDA | 재시도, DLQ, 재처리 절차를 문서화했는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 메시징·EDA | 토픽/파티션 키가 순서 보장 요건과 일치하는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 관측성(OTel+OpenSearch) | Trace, Metric, Log 수집 경로가 통합되어 있는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| 관측성(OTel+OpenSearch) | 핵심 비즈니스 트랜잭션에 Span/속성이 충분히 붙는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| 관측성(OTel+OpenSearch) | 로그와 트레이스를 Trace ID로 상호 연계할 수 있는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| 관측성(OTel+OpenSearch) | PII가 로그/트레이스에 노출되지 않도록 처리했는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| 관측성(OTel+OpenSearch) | SLO 기반 경보와 대시보드가 운영팀에 공유되는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| RAG | 문서 유형별 청킹 전략과 파라미터를 검증했는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| RAG | 임베딩 모델 선택 근거(정확도, 비용, 언어)가 있는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| RAG | 검색 전략(유사도, 하이브리드, MMR, 리랭킹)을 비교했는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| RAG | 근거 포함 응답과 환각 억제 프롬프트를 적용했는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| RAG | ragas 등 품질 지표를 주기적으로 측정하는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| 보안(Security) | TLS 1.3 강제, 보안 헤더, CORS 최소 허용 정책을 적용했는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 보안(Security) | OWASP Top 10 대응 통제와 담당자를 매핑했는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 보안(Security) | 비밀정보 저장소와 키 로테이션 주기를 운영하는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 보안(Security) | 취약점 스캔, SBOM, 이미지 서명을 릴리스 게이트에 넣었는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 보안(Security) | 감사 로그와 이상행위 탐지 규칙을 준비했는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 테스트 전략 | 단위/통합/E2E 비율과 실행 시점을 합의했는가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| 테스트 전략 | PR과 main 머지 시 필수 테스트 게이트가 자동 차단되는가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| 테스트 전략 | 외부 의존성은 Mock하고 DB 통합 테스트는 실DB를 사용하는가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| 테스트 전략 | 성능 테스트 기준(p95, 에러율, 동시성)을 수치로 관리하는가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| 테스트 전략 | 테스트 데이터/픽스처/팩토리 유지보수 책임이 명확한가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| Versions(버전 카탈로그) | 런타임, 프레임워크, 패키지 버전의 단일 출처를 유지하는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
| Versions(버전 카탈로그) | TBD 항목을 이슈로 관리하고 확정 기한을 두는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
| Versions(버전 카탈로그) | 문서 버전과 lockfile, 빌드 스크립트가 일치하는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
| Versions(버전 카탈로그) | 호환성 깨지는 업그레이드 시 마이그레이션 계획이 있는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
| Versions(버전 카탈로그) | 보안 패치 우선순위를 정해 정기 업데이트하는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
# Common Spec — 요소별 질문형 체크리스트 (상세)

`specs/edit/common` 각 요소를 리뷰할 때 사용하는 상세 점검표입니다.

| 항목 | 체크리스트 | 선택후부 | 미고려시 예상 상황 | 설명 |
|------|------------|----------|--------------------|------|
| ADR 포맷 | ADR 번호와 상태(제안/수락/폐기/대체)를 명확히 기록했는가?<br>배경에 제약사항(기술, 일정, 조직)을 구체적으로 적었는가?<br>검토한 옵션별 장단점이 비교 가능한 수준으로 작성됐는가?<br>선택한 옵션의 이유가 “왜” 중심으로 명시됐는가?<br>수락된 ADR을 수정하지 않고 새 ADR로 대체했는가? | `docs/adr/` 분리 관리, 단일 결정 문서 | 결정 근거 유실, 재논의 반복, 온보딩 지연 | 아키텍처 의사결정 이력 표준 |
| Agentic Agent | 목표 작업에 맞는 에이전트 패턴(ReAct, Plan-Execute 등)을 선택했는가?<br>`max_iterations`와 timeout을 설정해 무한 실행을 방지했는가?<br>파싱 오류 및 툴 실패 시 재시도/대체 흐름을 정의했는가?<br>툴 description이 입력/출력/사용 시점을 충분히 설명하는가?<br>트레이스/로그로 Thought-Action-Observation을 추적 가능한가? | ReAct, CoT, Plan-and-Execute, Multi-Agent | 무한 루프, 비용 급증, 불안정 응답 | LLM 에이전트 실행 안전장치 |
| API Gateway | 단일 진입점으로 TLS 종료와 라우팅을 일관되게 처리하는가?<br>JWT 검증 책임 경계를 게이트웨이와 백엔드에 명확히 나눴는가?<br>Rate limit 정책(분, 시간, 사용자 등급)을 수치로 정의했는가?<br>타임아웃, 서킷브레이커, 재시도 정책이 합의됐는가?<br>헬스체크/관리 경로가 외부 노출 정책과 분리되어 있는가? | Kong, Traefik, nginx+Lua, AWS API Gateway | 인증 불일치, DDoS 취약, 라우팅 혼선 | 외부 트래픽 제어 계층 |
| 인증·인가(Keycloak) | 프론트 채널에서 Auth Code + PKCE를 적용했는가?<br>JWT 서명, 만료, issuer, audience 검증을 모두 수행하는가?<br>Realm 역할과 Client 역할을 API 권한과 정확히 매핑했는가?<br>JWKS 캐시 갱신 주기와 키 로테이션 대응을 정의했는가?<br>브라우저나 공개 저장소에 client secret이 노출되지 않는가? | Keycloak Realm/Client, OAuth2 Resource Server, python-jose | 토큰 위변조, 권한 오작동, 시크릿 노출 | 통합 인증/권한 기준 |
| Backend (FastAPI) | 라우터, 서비스, 클라이언트, 스키마 계층이 분리되어 있는가?<br>Pydantic 제약조건과 `response_model`이 실제 계약과 일치하는가?<br>외부 호출에 timeout, retry, 에러 매핑을 적용했는가?<br>의존성 주입(`Depends`)으로 테스트 대체가 가능한 구조인가?<br>설정값과 시크릿이 환경별로 안전하게 분리되는가? | FastAPI, Uvicorn, httpx, tenacity | 입력 검증 누락, 장애 전파, 테스트 어려움 | Python API 서버 구현 기준 |
| Backend (Spring Boot) | Controller-Service-Client 책임이 명확히 분리되어 있는가?<br>`@Valid` 및 Bean Validation으로 입력 검증이 수행되는가?<br>공통 예외 처리(`@RestControllerAdvice`)로 에러 응답이 표준화됐는가?<br>WebClient/RestClient 설정(타임아웃, 풀, 재시도)이 공통화됐는가?<br>OpenAPI 문서와 실제 엔드포인트가 동기화되어 있는가? | Spring Boot, WebClient, RestClient, Virtual Threads | 응답 포맷 불일치, 검증 누락 | Java API 서버 구현 기준 |
| 브랜치 전략 | `main`/`develop` 보호 규칙(직접 push 금지, 필수 리뷰)을 설정했는가?<br>`feature/fix/release/hotfix` 네이밍 규칙을 팀에 공유했는가?<br>머지 전략(Squash/Rebase/Merge)과 커밋 메시지 기준을 정했는가?<br>핫픽스 후 develop 백포트 절차를 문서화했는가?<br>브랜치별 CI 필수 체크가 보호 규칙과 일치하는가? | GitFlow 유사 모델, trunk 기반 단순 모델 | 잘못된 브랜치 머지, 릴리스 혼선 | 코드 통합 경로 표준 |
| 비즈니스 스펙 포맷 | 문서가 “무엇을/왜” 중심으로 작성되고 “어떻게”를 배제했는가?<br>핵심 가치와 성공 기준이 측정 가능한 문장으로 표현됐는가?<br>주요 사용자 시나리오에 전제조건과 완료 기준이 있는가?<br>이해관계자 역할과 승인 책임이 명확히 적혀 있는가?<br>상태(초안/검토중/승인/폐기)와 버전이 최신인가? | 템플릿 기반 문서, Gherkin 보조 | 요구-구현 혼재, 검수 기준 부재 | 기능 요구 명세 품질 기준 |
| 캐싱 전략 | 캐시 계층(CDN, Gateway, Redis, DB)의 책임을 분리했는가?<br>키 네이밍 규칙에 서비스명/버전/엔티티가 포함되는가?<br>데이터 유형별 TTL과 갱신 트리거가 명시되어 있는가?<br>Cache-Aside 미스 처리와 무효화 이벤트를 정의했는가?<br>캐시 히트율과 stale 비율을 모니터링하는가? | Redis String, Hash, ZSet, CDN 캐시 | stale 데이터, 키 충돌, 비용 증가 | 성능/비용 최적화 기준 |
| 데이터 거버넌스 | 데이터 등급(L1~L4)과 PII 목록이 최신 상태인가?<br>수집 목적, 보존 기간, 삭제 방식이 데이터별로 정의됐는가?<br>로그/추적 데이터에 PII 마스킹이 적용되는가?<br>접근 권한이 최소 권한 원칙에 맞게 설정됐는가?<br>데이터 계보(출처→저장→가공→노출)를 추적할 수 있는가? | 필드 마스킹, OTel redaction, ILM | 개인정보 유출, 규제 위반 | 데이터 보호/준수 기준 |
| Database | PK, FK, ON DELETE 정책이 테이블별로 일관적인가?<br>정규화/비정규화 선택 사유가 문서와 주석에 남아 있는가?<br>쿼리 패턴 기준으로 인덱스를 설계하고 `EXPLAIN`으로 검증했는가?<br>`TIMESTAMPTZ`, JSONB 등 타입 선택 기준이 합의됐는가?<br>만료 데이터(TTL) 삭제 배치가 운영에 적용됐는가? | PostgreSQL, MySQL, SQLite, MongoDB | 무결성 저하, 성능 저하, 데이터 잔존 | 스키마/성능 설계 기준 |
| 배포 전략 | 동일 `git-sha` 이미지가 dev→staging→prod로 승격되는가?<br>환경별 승인자 수와 승인 조건이 명확한가?<br>E2E 실패 시 다음 환경 배포를 자동 차단하는가?<br>Canary 단계별 기준(에러율, 지연시간, 대기시간)이 정의됐는가?<br>롤백 명령과 담당자 연락체계를 사전에 점검했는가? | Rolling Update, Canary, Blue-Green(옵션) | 스테이징-프로덕션 불일치, 복구 지연 | 환경 승격/릴리스 운영 기준 |
| Frontend | 페이지별 특성에 맞게 CSR, SSR, SSG, Hybrid를 선택했는가?<br>로딩, 빈 상태, 오류, 재시도 UI가 정의되어 있는가?<br>요청 취소와 중복 요청 방지를 구현했는가?<br>상태 관리 범위(로컬/전역)와 도구 선택 근거가 있는가?<br>성능 지표(LCP, CLS 등)와 접근성 기준을 점검하는가? | Next.js, Vite, Zustand, Redux Toolkit | SEO 저하, UX 불안정 | 클라이언트 경험 설계 기준 |
| 인프라·CI/CD | `lint→test→build→image` 단계가 자동화되고 실패 시 차단되는가?<br>컨테이너가 멀티스테이지와 non-root 원칙을 따르는가?<br>Readiness/Liveness probe, HPA, 리소스 제한을 설정했는가?<br>환경별 ConfigMap/Secret과 Helm 값 분리가 되어 있는가?<br>취약점 스캔과 이미지 서명이 배포 전 수행되는가? | GitHub Actions, Helm, Kustomize, Vault | 배포 실패, 보안 취약, 환경 드리프트 | 운영 자동화 기준 |
| 메시징·EDA | 브로커 선택 기준(처리량, 순서, 보존, 운영성)을 정의했는가?<br>이벤트 스키마에 `event_id`, `version`, `correlation_id`가 있는가?<br>멱등성 키와 중복 소비 방지 로직이 구현되어 있는가?<br>재시도, DLQ, 재처리 절차를 문서화했는가?<br>토픽/파티션 키가 순서 보장 요건과 일치하는가? | Kafka, RabbitMQ, Redis Streams | 메시지 유실, 중복 처리 버그 | 비동기 아키텍처 기준 |
| 관측성(OTel+OpenSearch) | Trace, Metric, Log 수집 경로가 통합되어 있는가?<br>핵심 비즈니스 트랜잭션에 Span/속성이 충분히 붙는가?<br>로그와 트레이스를 Trace ID로 상호 연계할 수 있는가?<br>PII가 로그/트레이스에 노출되지 않도록 처리했는가?<br>SLO 기반 경보와 대시보드가 운영팀에 공유되는가? | OTel Collector, Jaeger/Tempo, Prometheus, OpenSearch | 원인 분석 지연, 블라인드 운영 | 장애 진단/성능 가시화 기준 |
| RAG | 문서 유형별 청킹 전략과 파라미터를 검증했는가?<br>임베딩 모델 선택 근거(정확도, 비용, 언어)가 있는가?<br>검색 전략(유사도, 하이브리드, MMR, 리랭킹)을 비교했는가?<br>근거 포함 응답과 환각 억제 프롬프트를 적용했는가?<br>ragas 등 품질 지표를 주기적으로 측정하는가? | Chroma, Pinecone, pgvector, Qdrant, bge/openai embedding | 환각 증가, 정확도 저하, 비용 폭증 | 검색증강 생성 품질 기준 |
| 보안(Security) | TLS 1.3 강제, 보안 헤더, CORS 최소 허용 정책을 적용했는가?<br>OWASP Top 10 대응 통제와 담당자를 매핑했는가?<br>비밀정보 저장소와 키 로테이션 주기를 운영하는가?<br>취약점 스캔, SBOM, 이미지 서명을 릴리스 게이트에 넣었는가?<br>감사 로그와 이상행위 탐지 규칙을 준비했는가? | cert-manager, Trivy, Dependabot, Cosign | 침해사고, 감사 실패 | 애플리케이션 보안 최소선 |
| 테스트 전략 | 단위/통합/E2E 비율과 실행 시점을 합의했는가?<br>PR과 main 머지 시 필수 테스트 게이트가 자동 차단되는가?<br>외부 의존성은 Mock하고 DB 통합 테스트는 실DB를 사용하는가?<br>성능 테스트 기준(p95, 에러율, 동시성)을 수치로 관리하는가?<br>테스트 데이터/픽스처/팩토리 유지보수 책임이 명확한가? | pytest, JUnit5, Playwright, k6, Testcontainers | 회귀 누락, 릴리스 품질 저하 | 품질 보증 실행 기준 |
| Versions(버전 카탈로그) | 런타임, 프레임워크, 패키지 버전의 단일 출처를 유지하는가?<br>TBD 항목을 이슈로 관리하고 확정 기한을 두는가?<br>문서 버전과 lockfile, 빌드 스크립트가 일치하는가?<br>호환성 깨지는 업그레이드 시 마이그레이션 계획이 있는가?<br>보안 패치 우선순위를 정해 정기 업데이트하는가? | 버전 표 문서, lockfile, 릴리스 노트 | 환경별 버전 불일치, 재현 실패 | 기술 스택 버전 기준선 |
