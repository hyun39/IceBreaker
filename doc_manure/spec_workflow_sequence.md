# Spec 워크플로우 시퀀스 다이어그램

## 전체 흐름 개요

```mermaid
sequenceDiagram
    actor User as 사용자
    participant Main as 메인 스레드<br/>(Claude Code)
    participant Loader as spec-system-prompt-loader
    participant Starter as spec-workflow-starter.md
    participant Req as spec-requirements<br/>(1~N개)
    participant Judge as spec-judge
    participant Design as spec-design<br/>(1~N개)
    participant Tasks as spec-tasks<br/>(1~N개)
    participant Impl as spec-impl
    participant Test as spec-test

    %% ── 0. 초기화 ──
    User->>Main: 기능 아이디어 설명
    Main->>Loader: 워크플로우 시작 요청
    Loader-->>Main: .claude/system-prompts/spec-workflow-starter.md 경로 반환
    Main->>Starter: 파일 읽기
    Starter-->>Main: 전체 워크플로우 지침 로드
    Main->>Main: feature_name 결정 (kebab-case)<br/>.claude/specs/{feature_name}/ 디렉토리 생성

    %% ── 1단계: 요구사항 ──
    Main->>User: "spec-requirements 에이전트를 몇 개 사용할까요? (1-128)"
    User-->>Main: N개 선택

    alt N = 1 (단일 실행)
        Main->>Req: 요구사항 생성 요청
        Req->>Req: EARS 형식 requirements.md 작성
        Req-->>Main: requirements.md 완성
    else N >= 2 (병렬 실행)
        par 병렬 생성
            Main->>Req: requirements_v1.md 생성 요청
            Main->>Req: requirements_v2.md 생성 요청
            Main->>Req: requirements_vN.md 생성 요청
        end
        Note over Main,Judge: 트리 기반 평가 시작
        Main->>Judge: N개 문서 평가 요청
        Judge->>Judge: 완성도/명확성/실현가능성/혁신성 채점
        Judge-->>Main: 최선안 선택 (requirements_v1234.md)
        Main->>Main: requirements_v1234.md → requirements.md 이름 변경
        Main->>Main: 평가 대상 임시 파일 삭제
    end

    Main->>User: requirements.md 검토 요청
    User-->>Main: 피드백 or 승인

    loop 사용자가 승인할 때까지
        User->>Main: 수정 요청
        Main->>Req: requirements.md 업데이트 요청
        Req-->>Main: 수정 완료
        Main->>User: 재검토 요청
        User-->>Main: 승인 ("좋아", "approved" 등)
    end

    %% ── 2단계: 설계 ──
    Main->>User: "spec-design 에이전트를 몇 개 사용할까요? (1-128)"
    User-->>Main: N개 선택

    alt N = 1 (단일 실행)
        Main->>Design: 설계 문서 생성 요청
        Design->>Design: requirements.md 읽기<br/>Mermaid 다이어그램 포함 design.md 작성
        Design-->>Main: design.md 완성
    else N >= 2 (병렬 실행)
        par 병렬 생성
            Main->>Design: design_v1.md 생성 요청
            Main->>Design: design_vN.md 생성 요청
        end
        Main->>Judge: N개 문서 평가 요청
        Judge-->>Main: 최선안 선택 → design.md
    end

    Main->>User: design.md 검토 요청

    loop 사용자가 승인할 때까지
        User->>Main: 수정 요청
        Main->>Design: design.md 업데이트
        Design-->>Main: 수정 완료
        Main->>User: 재검토 요청
        User-->>Main: 승인
    end

    %% ── 3단계: 태스크 ──
    Main->>User: "spec-tasks 에이전트를 몇 개 사용할까요? (1-128)"
    User-->>Main: N개 선택

    alt N = 1 (단일 실행)
        Main->>Tasks: 태스크 목록 생성 요청
        Tasks->>Tasks: requirements.md + design.md 읽기<br/>체크리스트 형식 tasks.md 작성
        Tasks-->>Main: tasks.md 완성
    else N >= 2 (병렬 실행)
        par 병렬 생성
            Main->>Tasks: tasks_v1.md 생성 요청
            Main->>Tasks: tasks_vN.md 생성 요청
        end
        Main->>Judge: N개 문서 평가 요청
        Judge-->>Main: 최선안 선택 → tasks.md
    end

    Main->>User: tasks.md 검토 요청

    loop 사용자가 승인할 때까지
        User->>Main: 수정 요청
        Main->>Tasks: tasks.md 업데이트
        Tasks-->>Main: 수정 완료
        Main->>User: 재검토 요청
        User-->>Main: 승인
    end

    %% ── 4단계: 구현 ──
    User->>Main: 태스크 실행 요청

    alt 기본 모드 (순차)
        loop 태스크 하나씩
            Main->>Impl: task_id 실행 요청
            Impl->>Impl: requirements/design/tasks.md 참조하여 코드 구현
            Impl->>Impl: tasks.md [ ] → [x] 마킹
            Impl-->>Main: 완료 보고
            Main->>User: 다음 태스크 진행 여부 확인
        end
    else 병렬 모드 (명시적 요청)
        par 의존성 없는 태스크 동시 실행
            Main->>Impl: task_2.1 실행
            Main->>Impl: task_2.2 실행
        end
        Impl-->>Main: 각 태스크 완료
    else 자동 모드 (전체 자동)
        Main->>Main: tasks.md 의존성 분석
        loop 의존성 순서대로
            par 독립 태스크 병렬 실행
                Main->>Impl: 독립 태스크 실행
            end
            Impl-->>Main: 완료
        end
    end

    %% ── 5단계: 테스트 (선택) ──
    opt 테스트 생성 요청 시
        User->>Main: "태스크 X.X 테스트 작성해줘"
        Main->>Test: task_id, feature_name 전달
        Test->>Test: 구현 코드 읽기<br/>{module}.md 테스트 케이스 문서 작성<br/>{module}.test.ts 실행 코드 작성
        Test-->>Main: 테스트 파일 완성
        Main->>User: 테스트 준비 완료 안내
    end
```

---

## spec-judge 트리 기반 평가 상세

병렬로 N개 문서가 생성된 경우, spec-judge는 다음 트리 구조로 평가합니다.

```mermaid
sequenceDiagram
    participant Main as 메인 스레드
    participant J1 as spec-judge #1
    participant J2 as spec-judge #2
    participant J3 as spec-judge #3
    participant JF as spec-judge (최종)

    Note over Main: 예: 10개 문서 생성됨

    par 1라운드 (3개 judge 병렬)
        Main->>J1: v1~v4 평가 요청
        J1-->>Main: v_1234.md (최선안)
        Main->>J2: v5~v7 평가 요청
        J2-->>Main: v_5678.md (최선안)
        Main->>J3: v8~v10 평가 요청
        J3-->>Main: v_9012.md (최선안)
    end

    Main->>JF: v_1234, v_5678, v_9012 최종 평가
    JF-->>Main: v_3456.md (최종 선택)

    Main->>Main: v_3456.md → requirements.md 이름 변경
    Main->>Main: 평가 대상 파일 삭제 (와일드카드 사용 금지)
```

---

## 생성 파일 위치

```
.claude/specs/{feature_name}/
├── requirements.md   ← 1단계 완료 후
├── design.md         ← 2단계 완료 후
└── tasks.md          ← 3단계 완료 후
```
