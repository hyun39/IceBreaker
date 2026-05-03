# Infinite Agentic Loop Q&A

참고 저장소: https://github.com/disler/infinite-agentic-loop

---

## Q1. 이 저장소의 명령어를 현재 프로젝트에서 쓰려면 어떻게 해?

이 프로젝트는 Claude Code의 커스텀 슬래시 명령어를 사용한다.  
현재 프로젝트에서 쓰려면 두 가지 방법이 있다.

**방법 1: 저장소 클론 후 직접 사용**
```bash
git clone https://github.com/disler/infinite-agentic-loop
cd infinite-agentic-loop
claude
```

**방법 2: 현재 프로젝트에 명령어 파일 복사**
```bash
cp -r infinite-agentic-loop/.claude/commands/ .claude/commands/
```

**주요 명령어 형식:**
```
/project:infinite <spec_file> <output_dir> <count>
```
- `count`: 숫자(1, 5, 20...) 또는 `infinite`

---

## Q2. `.claude/commands/` 에는 어떤 파일이 있어?

두 가지 슬래시 명령어 파일이 있다.

| 파일 | 역할 |
|------|------|
| `infinite.md` | 병렬 에이전트로 파일을 반복 생성하는 메인 명령어 |
| `prime.md` | 에이전트 루프 실행 전 컨텍스트를 준비하는 명령어 |

---

## Q3. `specs/` 폴더에는 뭐가 있어?

UI 컴포넌트 생성 스펙 파일 4개가 있다.

| 파일 | 설명 |
|------|------|
| `invent_new_ui.md` | 기존 UI 요소를 완전히 대체하는 새로운 컴포넌트 발명 |
| `invent_new_ui_v2.md` | 기존 컴포넌트를 애니메이션/인터랙션으로 크게 개선 |
| `invent_new_ui_v3.md` | 여러 UI 요소를 합친 테마형 하이브리드 컴포넌트 (단일 HTML) |
| `invent_new_ui_v4.md` | v3와 동일하지만 HTML/CSS/JS를 분리된 파일 구조로 생성 |

---

## Q4. 중복 작업 시 충돌 문제를 위해 output 폴더를 분리해야 할까?

**기본적으로 분리 불필요.** `infinite.md`는 내장 충돌 방지 메커니즘을 갖고 있다.

- PHASE 2에서 기존 파일 번호를 먼저 스캔
- 각 Sub Agent에게 `starting_number + agent_index`로 번호를 사전 배정 (실행 전에 미리 나눔)
- 오케스트레이터가 중복 번호 생성을 감시

```
Agent 1 → iteration_6.html
Agent 2 → iteration_7.html  ← 실행 전에 미리 배정
Agent 3 → iteration_8.html
```

**단, 아래 상황에서는 충돌 가능:**

| 상황 | 충돌 여부 |
|------|-----------|
| 하나의 `/project:infinite` 명령 내 병렬 실행 | 안전 (내장 조율) |
| `/project:infinite` 명령을 동시에 2개 실행 | 충돌 가능 |
| 같은 output 폴더에 시간차 없이 두 번 실행 | 충돌 가능 |

동시에 두 명령을 실행할 경우 폴더를 분리하는 것이 안전하다:
```
/project:infinite specs/invent_new_ui_v3.md src_batch_a 5
/project:infinite specs/invent_new_ui_v3.md src_batch_b 5
```

---

## Q5. 추가로 생성되는 작업 대상 파일명은 작업 전에 어떻게 식별해?

**두 가지 소스에서 파일명 패턴을 파악한다.**

**1. 스펙 파일에서 네이밍 패턴 확인 (PHASE 1)**

예: `invent_new_ui_v3.md`에는 아래가 명시되어 있다:
```
File Naming: ui_hybrid_[iteration_number].html
```

**2. output 폴더 스캔으로 최댓값 확인 (PHASE 2)**

기존 파일 중 가장 높은 번호를 찾아 +1부터 시작한다.

**전체 흐름:**
```
1. PHASE 1: 스펙 파일 읽기
   → 네이밍 패턴 확인: ui_hybrid_[iteration_number].html

2. PHASE 2: output 폴더 스캔
   → 기존 최댓값 확인: ui_hybrid_7.html → max = 7

3. PHASE 4: 에이전트 배정 전 번호 사전 계산
   → Agent 1 = ui_hybrid_8.html
   → Agent 2 = ui_hybrid_9.html  ← 실행 전에 이미 확정
   → Agent 3 = ui_hybrid_10.html
```

Sub Agent들은 자신이 써야 할 파일명을 처음부터 알고 시작하기 때문에 충돌이 방지된다.

---

## Q6. 오케스트레이터도 Sub Agent로 기동되는 걸까?

**아니다.** 오케스트레이터는 슬래시 명령을 받은 **메인 Claude 세션 자체**다.  
Sub Agent들만 `Task` 툴로 별도 spawning된다.

**구조:**
```
Claude Code 세션 (메인)
└── 오케스트레이터 ← /project:infinite 명령을 받아 직접 실행
    ├── Task tool → Sub Agent 1 (ui_hybrid_8.html 생성)
    ├── Task tool → Sub Agent 2 (ui_hybrid_9.html 생성)
    └── Task tool → Sub Agent 3 (ui_hybrid_10.html 생성)
```

오케스트레이터(메인 세션)의 컨텍스트는 Sub Agent들의 결과를 계속 받으면서 쌓이기 때문에,  
`infinite` 모드에서는 컨텍스트 한계에 도달하면 자연스럽게 종료된다.
