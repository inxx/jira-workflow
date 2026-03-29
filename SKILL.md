---
name: jira-ticket
description: Use when a user asks in Codex to create, update, assign, transition, sync, or template Jira tickets from WORK.md, ticket markdown, or existing Jira issues. Triggers on requests like "Jira 티켓 만들어줘", "서브태스크 추가", "WORK.md 업데이트", "Jira 싱크", "상태 변경", "작업자 등록", and "기존 Jira 티켓 보고 템플릿 만들어줘".
---

# Jira Ticket Management

## Core Rule

- `WORK.md` and ticket markdown are the source of truth.
- Jira is a writable sync target, not the authority.
- Unless the user explicitly asks to import from Jira, update Jira to match markdown, not the reverse.
- 기존 Jira 티켓을 템플릿으로 삼는 작업은 예외적인 bootstrap 모드다.
- bootstrap 모드에서도 Jira 한 건을 그대로 복사하지 말고, 재사용 가능한 구조만 추출한다.

## Codex-First Startup

스킬 실행 전 아래 순서로 프로젝트 규칙을 찾는다.

1. `AGENTS.md`
2. 리포지토리 문서 중 `*work-management*.md`, `*jira*.md`, `*workflow*.md`
3. `CLAUDE.md`
4. `.claude/rules/work-management.md`

여러 파일이 충돌하면 Codex용 문서인 `AGENTS.md`와 리포지토리 규칙 문서를 우선한다. 부족한 값만 사용자에게 물어본다.

### 필수 설정값

| 항목 | 설명 |
|------|------|
| Jira URL | 예: `https://company.atlassian.net` |
| Jira Cloud ID | MCP Jira 호출에 필요한 Cloud ID |
| Jira 프로젝트 키 | 예: `ABC` |
| Epic 코드 또는 Task 부모 규칙 | 예: `ABC-100` |
| `WORK.md` 경로 | 예: `WORK.md` |
| 티켓 MD 루트 | 예: `docs/work/` |

### 권장 설정값

| 항목 | 설명 |
|------|------|
| 팀 멤버 테이블 | 이름 -> 이메일 또는 Jira 계정 매핑 |
| 상태 매핑 표 | `WORK.md` 섹션 -> Jira 상태 또는 전환명 |

상태 매핑이 없으면 Jira 상태 전환은 보수적으로 처리한다. 섹션 이름만 보고 Jira workflow를 추정하지 않는다.

## 기준 파일 구조

```text
docs/work/
  TEMPLATE.md
  {epic-code}/
    {issue-key}-description.md
    {parent-key}/
      {subtask-key}-description.md
```

- 영구 파일명에는 실제 Jira 키만 사용한다.
- Jira 키를 받기 전에는 `{ticket-code}` 같은 placeholder 파일을 만들지 않는다.
- 필요하면 내용을 메모리에서 먼저 구성하거나 임시 slug 파일을 만들고, Jira 생성 직후 실제 키로 rename 한다.

### TEMPLATE.md 예시

```markdown
# {issue-key} {title}

## 작업자

## 배경

## 요구사항

## 하위 작업

- [ ] {subtask-description}

## 관련 파일

## 참고 자료

## 완료 조건
```

## WORK.md 규칙

### Task 항목

```text
- 항목 설명 @작업자
- 항목 설명 @작업자 {issue-key} [detail](docs/work/{epic-code}/{issue-key}-description.md)
```

### Subtask 항목

```text
  - [ ] 설명 {subtask-key} [detail](docs/work/{epic-code}/{parent-key}/{subtask-key}-description.md)
  - [ ] 설명 (no ticket)
  - [x] 완료된 항목
```

### 기본 섹션 순서

```text
## TODO
## DOING
## IN REVIEW
## IN QA
## DONE
## DROP
```

- 이 섹션은 로컬 작업 레인이다.
- Jira workflow 이름과 1:1로 같다고 가정하지 않는다.
- Jira 키가 없는 항목은 로컬 backlog일 수 있으니, 사용자가 요청하기 전에는 자동 생성하지 않는다.

## Jira MCP 도구

| 도구 | 용도 |
|------|------|
| `createJiraIssue` | 신규 이슈 생성 |
| `editJiraIssue` | description, assignee 업데이트 |
| `getJiraIssue` | 현재 상태와 필드 조회 |
| `transitionJiraIssue` | 상태 전환 |
| `lookupJiraAccountId` | 이름 또는 이메일 -> account ID 변환 |

## 워크플로우

### 1. Task 생성

```text
Step 1: 컨텍스트 읽기
  - repo 규칙, WORK.md, 인접 티켓 문서를 읽는다
  - 제목, 배경, 요구사항, 완료 조건, 작업자를 정리한다

Step 2: Jira Task 생성
  - createJiraIssue 호출로 실제 Jira 키를 먼저 받는다
  - issueTypeName: "Task"
  - parent: {epic-code} 또는 프로젝트 규칙의 Task 부모 규칙
  - summary: 티켓 제목
  - description: 배경 + 요구사항 + 완료 조건

Step 3: 최종 MD 파일 작성
  - 경로: docs/work/{epic-code}/{issue-key}-description.md
  - 제목: # {issue-key} {title}
  - 참고 자료에 Jira 링크 추가

Step 4: WORK.md 반영
  - ## TODO 섹션에 항목 추가 또는 기존 초안 항목 업데이트
  - 형식: - 설명 @작업자 {issue-key} [detail](...)

Step 5: 작업자 싱크
  - 작업자가 명시된 경우 lookupJiraAccountId -> editJiraIssue 순으로 반영
```

### 2. Subtask 생성

```text
Step 1: 부모 컨텍스트 확인
  - 부모 티켓 코드와 부모 MD 파일을 확인한다
  - 서브태스크 제목, 요구사항, 완료 조건을 정리한다

Step 2: Jira Subtask 생성
  - createJiraIssue 로 실제 subtask 키를 먼저 받는다
  - issueTypeName: "Subtask"
  - parent: {parent-ticket-code}

Step 3: 최종 MD 파일 작성
  - 경로: docs/work/{epic-code}/{parent-key}/{subtask-key}-description.md
  - 제목과 Jira 링크를 실제 키로 기록한다

Step 4: 부모 MD와 WORK.md 갱신
  - 부모 MD의 ## 하위 작업에 코드와 링크 추가
  - WORK.md 부모 task 아래에 들여쓰기된 checkbox 항목 추가

Step 5: 작업자 싱크
  - 요청에 작업자가 있으면 Jira assignee까지 동기화한다
```

### 3. 설명, 작업자, 링크 갱신

```text
Step 1: MD와 WORK.md를 먼저 수정한다
Step 2: Jira description / assignee를 markdown 기준으로 editJiraIssue 한다
Step 3: 링크와 파일 경로가 실제 Jira 키와 일치하는지 확인한다
```

### 4. 상태 변경

```text
Step 1: WORK.md에서 항목을 목표 섹션으로 이동한다
Step 2: getJiraIssue 로 현재 Jira 상태를 읽는다
Step 3: 프로젝트 설정의 상태 매핑 표로 Jira 목표 상태 또는 전환명을 결정한다
Step 4: 현재 상태와 목표가 다를 때만 transitionJiraIssue 를 호출한다
Step 5: 매핑이 없거나 모호하면 WORK.md만 옮기고 Jira 전환은 보류했다고 보고한다
```

- `TODO -> DOING -> IN REVIEW` 같은 문자열을 Jira 전환명으로 그대로 쓰지 않는다.
- Codex에서는 "현재 사용자"를 안정적으로 알 수 없으니, 상태 변경만으로 작업자를 자동 지정하지 않는다.

### 5. 전체 싱크

```text
Step 1: WORK.md, 관련 티켓 MD, Jira 현재 상태를 함께 읽는다
Step 2: 아래 항목을 비교한다
  - summary / description
  - assignee
  - status
  - subtask 목록
Step 3: markdown -> Jira 방향으로 동기화 가능한 항목만 반영한다
Step 4: 상태 매핑이나 작업자 매핑이 없으면 해당 항목은 건너뛰고 blocker를 보고한다
Step 5: (no ticket) 항목은 자동 생성하지 않고 결과에만 포함한다
```

### 6. 기존 Jira 티켓 기반 템플릿 생성

이 워크플로우는 사용자가 명시적으로 요청할 때만 실행한다.

```text
Step 1: bootstrap 범위 확인
  - 사용자가 기준 티켓 키를 줬는지 확인한다
  - 가능하면 같은 프로젝트/같은 Epic/같은 issue type의 티켓 3~5개를 기준으로 삼는다

Step 2: 기존 티켓 수집
  - getJiraIssue 로 각 티켓의 summary, description, status, assignee, 하위 작업 구조를 읽는다
  - 로컬에 대응하는 ticket markdown이 있으면 함께 읽는다

Step 3: 공통 패턴 추출
  - 제목 패턴
  - description 섹션 제목과 순서
  - 하위 작업 작성 방식
  - 관련 링크, 완료 조건, 참고 자료 같은 반복 섹션

Step 4: 템플릿 초안 작성
  - `docs/work/TEMPLATE.md` 또는 프로젝트 템플릿 파일을 새로 만들거나 갱신한다
  - 티켓 고유 정보 대신 placeholder를 넣는다
  - 사람 이름, 날짜, 특정 이슈 코드, 일회성 메모는 제거한다

Step 5: 프로젝트 규칙 후보 정리
  - 반복적으로 보이는 상태 흐름, 담당자 표기, 링크 규칙이 있으면 `AGENTS.md` 후보로 제안한다
  - Jira custom field는 도구 지원이 불명확하면 템플릿 본문에 강제하지 않는다

Step 6: 결과 보고
  - 어떤 티켓을 참고했는지
  - 공통 규칙으로 채택한 내용
  - 의도적으로 제외한 티켓 고유 정보
  - 표본이 1개뿐이면 신뢰도가 낮다고 명시한다
```

- 기본 모드에서는 Jira를 템플릿 원본으로 보지 않는다.
- bootstrap 모드에서도 Jira 내용을 그대로 정답으로 취급하지 말고, 팀이 반복해서 쓰는 형식만 추출한다.
- 서로 다른 프로젝트나 workflow의 티켓을 섞어 읽으면 템플릿이 오염될 수 있으니 가능하면 같은 묶음만 사용한다.
- 사용자가 특정 티켓 1개만 주면 seed 예시로만 사용하고, 가능한 경우 추가 티켓을 더 요청하거나 주변 티켓을 찾아 비교한다.

## 작업자 싱크 규칙

```text
1. 요청 본문 또는 MD의 ## 작업자 섹션에서 이름을 읽는다
2. 팀 멤버 테이블에서 이메일 또는 Jira 계정을 찾는다
3. lookupJiraAccountId(email or account) -> accountId
4. editJiraIssue({ assignee: { accountId } })
```

- 팀 테이블에서 유일하게 매칭되지 않으면 사용자에게 확인한다.
- 로컬 git 사용자명, 시스템 계정, 대화 상대 이름을 근거로 assignee를 추정하지 않는다.

## 상태 매핑 예시

프로젝트 문서에 아래처럼 명시해 두면 안전하다.

```markdown
## Jira status mapping

| WORK.md | Jira target |
|---------|-------------|
| TODO | To Do |
| DOING | In Progress |
| IN REVIEW | In Review |
| IN QA | QA |
| DONE | Done |
| DROP | Cancelled |
```

`Jira target`은 상태명 또는 실제 전환명 중 프로젝트에서 쓰는 값을 그대로 적는다.

## 결과 보고 형식

```text
JIRA TICKET: [CREATED/UPDATED/SYNCED]

티켓:     ABC-123 - {title}
상태:     TODO -> DOING
작업자:   홍길동 (synced)
MD:       docs/work/ABC-100/ABC-123-description.md
Jira:     https://company.atlassian.net/browse/ABC-123
```

## Claude Code 호환 메모

- 이 스킬은 Codex 기준으로 작성되었다.
- Claude Code에서 사용할 때도 같은 규칙을 따르되, Codex 문서가 없을 때만 `CLAUDE.md`와 `.claude/rules/...`를 기본 설정 소스로 사용한다.
