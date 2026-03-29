# jira-workflow

Codex에서 `WORK.md`와 티켓 markdown을 기준으로 Jira 티켓을 만들고, 담당자를 맞추고, 상태를 동기화하도록 돕는 스킬입니다.

쉽게 말하면:

- 작업 계획은 `WORK.md`에 적고
- 자세한 내용은 티켓 markdown에 적고
- Codex가 그 내용을 바탕으로 Jira를 생성하거나 업데이트합니다

Jira를 직접 클릭하면서 반복 작업하는 대신, 평소처럼 문서를 쓰고 Codex에게 요청하는 방식에 가깝습니다.

## 이 스킬이 하는 일

- Jira Task / Subtask 생성
- Jira 키를 받은 뒤 티켓 markdown 파일 생성
- `WORK.md`와 티켓 문서를 기준으로 Jira description 동기화
- 팀 멤버 표를 이용한 assignee 자동 매핑
- `WORK.md` 섹션과 Jira 상태를 매핑해서 상태 변경
- 기존 Jira 티켓을 읽어 `TEMPLATE.md` 초안 만들기
- 내게 할당된 Jira 티켓을 `docs/work`로 import

## 이런 사람에게 맞습니다

- Jira 티켓을 문서 기반으로 관리하고 싶은 팀
- `WORK.md`를 작업의 기준 문서로 쓰는 팀
- "티켓 만들어줘", "상태 옮겨줘", "담당자 지정해줘"를 Codex에게 맡기고 싶은 사람
- 팀이 이미 쓰고 있던 Jira 티켓 스타일을 템플릿으로 정리하고 싶은 사람

## 3분 시작

### 1. 스킬 설치

아래 명령으로 이 폴더를 Codex 스킬 폴더에 연결합니다.

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$(pwd)" "${CODEX_HOME:-$HOME/.codex}/skills/jira-ticket"
```

설치가 끝나면 Codex는 이 스킬을 자동으로 사용할 수 있습니다. 필요하면 요청에 `$jira-ticket`를 직접 적어서 명시적으로 부를 수도 있습니다.

예:

```text
Use $jira-ticket to create a Jira task from WORK.md
```

### 2. Jira MCP 먼저 연결

이 스킬은 Jira에 직접 접속하지 않습니다. Codex에 연결된 `Jira MCP 서버`를 통해 Jira를 읽고 씁니다.

즉, 순서는 아래처럼 이해하면 됩니다.

- `jira-ticket` 스킬: 작업 규칙과 문서 동기화 방법 담당
- Jira MCP: 실제 Jira API 연결과 인증 담당

둘 다 있어야 제대로 동작합니다.

#### Jira MCP에서 보통 필요한 정보

Jira MCP를 Codex에 연결할 때는 보통 아래 종류의 정보가 필요합니다.

- Jira 사이트 URL
- Atlassian Cloud ID
- Jira 계정 인증 정보

중요:

- 계정 토큰, 비밀번호, OAuth 같은 인증 정보는 `AGENTS.md`에 적지 않습니다
- 이런 민감 정보는 Jira MCP 설정 단계에서만 입력합니다
- 이 README는 특정 Jira MCP 구현체의 설치 명령까지 강제하지 않습니다

이유는 Jira MCP 서버마다 설치 방식이 다를 수 있기 때문입니다. 어떤 환경은 앱 UI에서 MCP를 추가하고, 어떤 환경은 설정 파일에 MCP 서버를 등록합니다.

처음 연결할 때 Codex에서 아래처럼 확인해보면 됩니다.

```text
Jira 연결이 되는지 확인해줘. 새 티켓은 만들지 말고 Jira MCP와 설정만 점검해줘.
```

### 3. 프로젝트에 최소 설정 추가

리포지토리 루트의 `AGENTS.md`에 아래 내용을 넣는 것을 권장합니다.

처음부터 직접 쓰기보다, 이 레포의 [AGENTS.example.md](AGENTS.example.md)를 복사해서 시작하는 편이 가장 쉽습니다.

```bash
cp AGENTS.example.md AGENTS.md
```

```markdown
## Jira workflow

- Jira URL: https://your-company.atlassian.net
- Jira Cloud ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
- Jira 프로젝트 키: ABC
- Epic 코드: ABC-100
- WORK.md 경로: WORK.md
- 티켓 MD 루트: docs/work/
- 기본 Jira 사용자: hong@company.com
```

앞의 6개가 최소 설정입니다. 마지막 `기본 Jira 사용자`는 `"나에게 할당된 티켓"` import 기능을 쓸 때 특히 유용한 권장값입니다.

즉, 최소 시작은 아래 6개입니다.

- Jira URL
- Jira Cloud ID
- Jira 프로젝트 키
- Epic 코드
- `WORK.md` 경로
- 티켓 MD 루트

#### 이 정보는 왜 또 필요한가요?

Jira MCP가 이미 연결되어 있어도, 그건 "Jira에 접속할 수 있다"는 뜻일 뿐입니다.

이 스킬은 추가로 아래 정보를 알아야 합니다.

- 어떤 Jira 프로젝트에 티켓을 만들지
- 어떤 Epic 아래에 Task를 만들지
- `WORK.md`가 어디 있는지
- 티켓 문서를 어느 폴더에 저장할지
- `"나에게 할당된 티켓"` 요청에서 누구를 기준으로 볼지

즉:

- Jira MCP 설정: 접속과 인증
- `AGENTS.md` 설정: 프로젝트 운영 규칙

이 둘은 역할이 다릅니다.

### 4. 있으면 좋은 설정도 추가

담당자 자동 지정과 상태 동기화를 쓰려면 아래도 같이 넣는 게 좋습니다.

```markdown
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

이 표가 있으면:

- `홍길동` 같은 이름을 Jira assignee로 바꿔 넣을 수 있고
- `WORK.md`의 `DOING`, `DONE` 같은 섹션을 Jira 상태와 안전하게 연결할 수 있습니다

### 5. 첫 요청 해보기

설치와 설정이 끝났다면 Codex에게 아래처럼 말해보면 됩니다.

```text
결제 기능 Jira 티켓 만들어줘
```

또는 좀 더 구체적으로:

```text
WORK.md 기준으로 결제 기능 task를 Jira에 만들고 docs/work에도 문서 생성해줘
```

정상적으로 동작하면 Codex는 보통 이런 일을 합니다.

- `WORK.md`와 관련 문서를 읽음
- Jira에 Task 생성
- 발급된 Jira 키로 티켓 markdown 파일 생성
- `WORK.md` 항목에 Jira 키와 detail 링크 반영

## 어떤 정보를 어디에 입력해야 하나요?

처음 쓰는 사람이 가장 헷갈리는 부분이라, 아래처럼 나눠서 보면 쉽습니다.

### 1. Jira MCP 쪽에 넣는 정보

여기는 Jira 접속을 위한 정보입니다.

- Jira 사이트 URL
- Atlassian Cloud ID
- Jira 계정 인증 정보

이 정보는 Codex의 MCP 연결 설정에 넣습니다. `README.md`, `SKILL.md`, `AGENTS.md` 같은 프로젝트 문서에 토큰이나 비밀번호를 적지 않는 것이 원칙입니다.

### 2. `AGENTS.md`에 넣는 정보

여기는 이 프로젝트에서 스킬이 어떻게 동작해야 하는지 알려주는 정보입니다.

- Jira URL
- Jira Cloud ID
- Jira 프로젝트 키
- Epic 코드
- `WORK.md` 경로
- 티켓 MD 루트
- 기본 Jira 사용자
- 작업자 표
- 상태 매핑 표

이 정보는 민감한 인증값이 아니라, 프로젝트 규칙과 경로 정보에 가깝습니다.

### 3. 매번 직접 입력해야 하나요?

보통은 아닙니다.

아래 두 가지가 준비되어 있으면, 매 요청마다 반복 입력할 필요가 거의 없습니다.

- Codex에 Jira MCP가 연결되어 있음
- 리포지토리에 `AGENTS.md`가 있음

이 상태라면 사용자는 보통 아래처럼 짧게만 말하면 됩니다.

```text
결제 기능 Jira 티켓 만들어줘
```

```text
ABC-812 상태를 DOING으로 옮겨줘
```

### 4. 언제 직접 입력이 필요한가요?

아래 경우에는 Codex가 추가 정보를 물을 수 있습니다.

- Jira MCP가 아직 연결되지 않음
- `AGENTS.md`가 없음
- 프로젝트 키나 Epic 코드가 빠져 있음
- `기본 Jira 사용자`가 없어 "나"를 Jira 사용자로 해석할 수 없음
- 작업자 이름이 팀 멤버 표에 없음
- 상태 매핑이 없어 Jira 상태를 확정할 수 없음

이론상 모든 정보를 요청 문장에 직접 써서 한 번만 실행할 수도 있지만, 계속 그렇게 쓰는 것은 비추천입니다.

예를 들면 이런 식입니다.

```text
Jira URL은 https://your-company.atlassian.net 이고 프로젝트 키는 ABC, Epic은 ABC-100, WORK.md는 루트, 문서 경로는 docs/work 기준으로 결제 기능 티켓 만들어줘
```

이렇게도 가능은 하지만, `AGENTS.md` 한 번 작성해두는 편이 훨씬 편합니다.

## 처음 쓰는 사람이 자주 하는 질문

### Q. 꼭 `AGENTS.md`가 있어야 하나요?

있는 것이 가장 좋습니다. 이 스킬은 `AGENTS.md`를 가장 먼저 읽도록 설계되어 있습니다. 없으면 다른 workflow 문서나 `CLAUDE.md`도 참고할 수 있지만, 설정 위치가 흩어지면 처음 쓰는 사람이 헷갈리기 쉽습니다.

### Q. Jira MCP만 연결하면 바로 되나요?

아직 아닙니다. Jira MCP만 연결되어 있으면 "Jira에 접속"은 가능하지만, 이 프로젝트에서 어느 경로를 쓰고 어느 프로젝트에 티켓을 만들지까지는 모릅니다. 그래서 `AGENTS.md` 같은 프로젝트 설정이 같이 필요합니다.

### Q. 반대로 `AGENTS.md`만 있으면 되나요?

이것도 아닙니다. `AGENTS.md`만 있고 Jira MCP가 없으면 문서 규칙은 알 수 있어도 Jira에 실제 생성, 수정, 상태 전환을 할 수 없습니다.

### Q. 인증 정보도 `AGENTS.md`에 적어야 하나요?

아니요. 인증 토큰, 비밀번호, OAuth 정보는 Jira MCP 설정에만 넣고, 프로젝트 문서에는 적지 않는 것을 권장합니다.

### Q. "나에게 할당된 티켓 가져와줘"도 되나요?

됩니다. 다만 안전하게 처리하려면 `AGENTS.md`에 `기본 Jira 사용자`를 적어두는 편이 좋습니다. 예를 들어 `hong@company.com`처럼 내 Jira 이메일이나 계정을 넣어두면 Codex가 그 사용자를 기준으로 import 할 수 있습니다.

### Q. import하면 기존 문서를 덮어쓰나요?

아니요. 기본 동작은 보수적입니다. 같은 Jira 키 문서가 이미 있으면 덮어쓰기보다 sync 후보로 분류하는 쪽을 우선합니다.

### Q. `WORK.md`가 아직 없으면요?

먼저 `WORK.md`를 만드는 편이 좋습니다. 이 스킬은 `WORK.md`를 작업 기준 문서로 가정합니다.

최소 예시는 아래처럼 시작할 수 있습니다.

```markdown
## TODO

- 결제 기능 구현 홍길동

## DOING

## IN REVIEW

## IN QA

## DONE

## DROP
```

### Q. 티켓 문서 파일은 언제 만들어지나요?

이 스킬은 Jira 키를 먼저 받은 뒤 최종 파일명을 확정합니다. 즉, 아직 발급되지 않은 티켓 번호를 미리 파일명으로 쓰지 않습니다.

### Q. Jira 상태가 우리 팀 이름과 다르면요?

그래서 `Jira status mapping` 표를 두는 것을 권장합니다. `WORK.md`의 섹션 이름과 Jira 상태 이름이 다를 수 있기 때문입니다.

## 연결 확인 방법

이 스킬이 제대로 동작하려면 Jira MCP 서버 연결이 필요합니다.

연결이 안 되어 있으면 보통 이런 문제가 생깁니다.

- Jira 이슈 생성 단계에서 실패함
- assignee 변경이 안 됨
- 상태 전환이 안 됨
- Codex가 Jira 관련 도구를 사용할 수 없다고 말함

가장 쉬운 확인 방법은 Codex에게 작은 요청을 해보는 것입니다.

```text
Jira 연결이 되는지 확인해줘. 새 티켓은 만들지 말고 필요한 설정만 점검해줘.
```

이 요청으로 Codex가 설정 파일과 Jira 도구 사용 가능 여부를 먼저 확인하게 만들 수 있습니다.

함께 점검하면 좋은 항목은 아래입니다.

- Jira MCP 도구가 실제로 보이는지
- `AGENTS.md`에 필수 값이 있는지
- Jira 프로젝트 키와 Epic 코드가 있는지
- `WORK.md` 경로와 티켓 문서 경로가 존재하는지

## 자주 쓰는 요청 예시

```text
결제 기능 Jira 티켓 만들어줘
```

```text
홍길동 담당으로 서브태스크 추가해줘
```

```text
ABC-812 상태를 DOING으로 옮겨줘
```

```text
WORK.md 기준으로 Jira 싱크해줘
```

```text
이 이슈 설명을 docs/work 문서 기준으로 업데이트해줘
```

```text
나에게 할당된 Jira 티켓 가져와서 docs/work에 넣어줘
```

```text
홍길동에게 할당된 Jira 티켓만 import 해줘
```

```text
기존 Jira 티켓 ABC-101, ABC-104, ABC-108을 보고 TEMPLATE.md 초안 만들어줘
```

```text
기존 Jira 티켓 하나를 기준으로 새 티켓 템플릿 형태로 정리해줘
```

## 기존 Jira 티켓으로 템플릿 만들기

팀이 이미 Jira를 꽤 써왔는데 `WORK.md` + markdown 방식은 이제 시작하는 경우에 특히 유용합니다.

이때는 기존 Jira 티켓 몇 개를 읽고 공통 형식만 추출해서 `TEMPLATE.md` 초안을 만들 수 있습니다.

추천 방식은 아래와 같습니다.

- 같은 프로젝트의 티켓 사용
- 가능하면 같은 Epic 또는 비슷한 종류의 티켓 3~5개 사용
- 한 장짜리 티켓 복붙이 아니라 공통 섹션만 추출

예를 들어 아래처럼 요청하면 됩니다.

```text
기존 Jira 티켓 3개를 보고 우리 팀 스타일에 맞는 TEMPLATE.md 초안 만들어줘
```

주의할 점도 있습니다.

- 서로 다른 프로젝트의 티켓을 섞으면 템플릿이 이상해질 수 있습니다
- 사람 이름, 일정, 특정 이슈 코드 같은 값은 템플릿에서 제거하는 편이 좋습니다
- 티켓 1개만 기준으로 만들면 품질이 떨어질 수 있습니다

## 내 할당 티켓 가져오기

이 기능은 Jira에 이미 있는 티켓을 로컬 작업 문서로 옮겨와서, 이후부터 `WORK.md`와 markdown 기준으로 관리하고 싶을 때 유용합니다.

보통은 아래처럼 요청합니다.

```text
나에게 할당된 Jira 티켓 가져와서 docs/work에 넣어줘
```

더 구체적으로도 가능합니다.

```text
나에게 할당된 진행중 티켓만 가져와서 WORK.md까지 반영해줘
```

추천 조건은 아래와 같습니다.

- `AGENTS.md`에 `기본 Jira 사용자`가 적혀 있음
- 상태 매핑 표가 있음
- Jira MCP에 issue search 또는 JQL 조회 기능이 있음

동작 방식은 보수적입니다.

- 이미 로컬에 있는 티켓 문서는 덮어쓰지 않음
- 완료된 티켓은 기본적으로 제외
- 상태 매핑이 불명확하면 문서만 만들고 `WORK.md` 반영은 보류
- Epic 경로를 모르면 `docs/work/imported/` 아래에 먼저 저장

만약 Jira MCP가 검색 도구를 제공하지 않으면, 자동 목록 조회 대신 티켓 키를 직접 넘겨 import 하는 방식으로 진행할 수 있습니다.

## 권장 폴더 구조

```text
WORK.md
AGENTS.md
docs/
  work/
    TEMPLATE.md
    imported/
      ABC-999-description.md
    ABC-100/
      ABC-123-description.md
      ABC-123/
        ABC-124-description.md
```

모든 팀이 정확히 이 구조일 필요는 없지만, `AGENTS.md`에 경로만 정확히 적어두면 Codex가 그 규칙을 따라갑니다.

## Claude Code에서 쓸 때

이 스킬은 Codex 기준으로 작성되었습니다. Claude Code에서 같이 쓰고 싶다면 보조적으로 아래처럼 연결할 수 있습니다.

```bash
ln -s "$(pwd)/SKILL.md" ~/.claude/skills/jira-ticket.md
```

다만 설정 문서는 가능하면 `AGENTS.md`를 기준으로 유지하는 편이 덜 헷갈립니다.

## 더 자세한 규칙

스킬 내부 동작 규칙은 [SKILL.md](SKILL.md)에 정리되어 있습니다. README는 사람용 빠른 안내서이고, `SKILL.md`는 Codex가 따르는 작업 규칙 문서입니다.
