# AGENTS Example

이 파일을 `AGENTS.md`로 복사한 뒤, 예시 값을 실제 프로젝트 값으로 바꿔서 사용하세요.

```markdown
## Jira workflow

- Jira URL: https://your-company.atlassian.net
- Jira Cloud ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
- Jira 프로젝트 키: ABC
- Epic 코드: ABC-100
- WORK.md 경로: WORK.md
- 티켓 MD 루트: docs/work/

## 작업자

| 이름   | Jira 계정        |
| ------ | ---------------- |
| 홍길동 | hong@company.com |
| 김영희 | younghee@company.com |

## Jira status mapping

| WORK.md | Jira target |
| ------- | ----------- |
| TODO | To Do |
| DOING | In Progress |
| IN REVIEW | In Review |
| IN QA | QA |
| DONE | Done |
| DROP | Cancelled |
```

## 작성 가이드

- `Jira URL`: 회사 Jira 주소
- `Jira Cloud ID`: Jira MCP에서 사용하는 Atlassian Cloud ID
- `Jira 프로젝트 키`: 티켓을 만들 프로젝트 키
- `Epic 코드`: Task를 연결할 Epic 키. 프로젝트마다 다른 parent 규칙을 쓰면 그 규칙에 맞춰 적으세요.
- `WORK.md 경로`: 루트가 아니면 실제 상대 경로로 변경
- `티켓 MD 루트`: 티켓 문서를 생성할 폴더

## 사용 순서

1. 이 파일을 `AGENTS.md`로 복사
2. 예시 값을 실제 값으로 교체
3. 필요한 경우 팀 멤버 표와 상태 매핑 표 보완
4. Codex에게 `Jira 연결이 되는지 확인해줘. 새 티켓은 만들지 말고 설정만 점검해줘.` 라고 요청
