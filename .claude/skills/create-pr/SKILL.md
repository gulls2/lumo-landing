---
name: create-pr
description: 현재 브랜치의 변경사항을 분석해 GitHub Pull Request를 생성한다. 프로젝트 성격에 따라 영문 또는 한국어 PR 템플릿을 선택한다. "PR 만들어줘", "PR 생성", "풀리퀘스트 올려줘", "pull request 열어줘", "create a PR" 처럼 PR 생성을 요청하면 반드시 이 스킬을 사용한다. 커밋 직후 "이제 올려줘"처럼 PR을 암시하는 요청에도 사용한다.
context: fork
argument-hint: "[base 브랜치] [--draft] [--lang en|ko]"
---

# create-pr

현재 브랜치를 base 브랜치로 향하는 PR로 만든다. 이 스킬은 `context: fork`로 실행되어 **서브 에이전트가 대화 맥락 없이** 동작한다. 필요한 정보는 전부 git과 저장소에서 직접 수집한다. 사용자에게 되묻지 말고 아래 규칙대로 판단한다.

인자(`$ARGUMENTS`): base 브랜치 이름, `--draft`, `--lang en|ko`를 받을 수 있다. 없으면 기본값을 쓴다.

## 1. 사전 점검

- `git rev-parse --is-inside-work-tree`로 저장소 여부를 확인한다. 아니면 중단하고 이유를 보고한다.
- `gh auth status`로 GitHub 인증을 확인한다. 실패하면 중단하고 `gh auth login`이 필요하다고 보고한다.
- `git remote -v`에 원격이 없으면 중단하고 보고한다. 원격 저장소를 임의로 만들지 않는다.
- 현재 브랜치가 base 브랜치(`main`/`master`)와 같으면 중단한다. PR은 작업 브랜치에서만 만든다.
- 커밋되지 않은 변경사항이 있으면 PR 본문에 포함되지 않는다는 점을 최종 보고에 적는다. 대신 커밋하지 않는다. 커밋은 `commit` 스킬의 일이다.

## 2. base 브랜치 결정

우선순위: 인자 → `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` → `main` → `master`.

## 3. 변경사항 분석

- `git log <base>..HEAD --oneline`으로 커밋 목록을, `git diff <base>...HEAD --stat`과 `git diff <base>...HEAD`로 실제 변경을 읽는다.
- 커밋 메시지만 믿지 말고 diff를 읽어서 "무엇이 왜 바뀌었는지"를 파악한다. 리뷰어는 diff를 보면 알 수 있는 내용이 아니라 **의도와 영향 범위**를 알고 싶어 한다.
- 커밋이 0개면 PR 대상이 없으므로 중단하고 보고한다.

## 4. 템플릿 선택

**저장소 자체 템플릿이 최우선이다.** `.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `docs/pull_request_template.md`, 루트의 `pull_request_template.md`가 있으면 그것을 그대로 채운다. 이 경우 아래 두 템플릿은 쓰지 않는다. 팀이 정한 형식을 어기면 CI나 리뷰 규칙과 충돌할 수 있기 때문이다.

자체 템플릿이 없으면 언어를 정한다. `--lang` 인자가 있으면 그대로 따르고, 없으면 아래 신호를 **위에서부터** 보고 처음 확실한 신호로 결정한다.

| 순서 | 신호 | 판정 |
|------|------|------|
| 1 | `CLAUDE.md`·`AGENTS.md`·`CONTRIBUTING.md`에 PR/커밋 언어 규정이 있음 | 규정을 따른다 |
| 2 | 최근 PR 제목·본문(`gh pr list --state all --limit 10 --json title,body`)의 언어 | 다수 언어를 따른다 |
| 3 | 최근 커밋 메시지(`git log -n 20 --format=%s`)의 언어 | 다수 언어를 따른다 |
| 4 | `README.md`의 주 언어, `lang="ko"` 등 프로젝트 로케일 | 그 언어를 따른다 |
| 5 | 저장소 소유자가 한국 조직·개인으로 보이는 경우(README·문서에 한국어) | 한국어 |

- 한국어 프로젝트 → `references/template-ko.md`
- 해외 오픈소스 등 영어 프로젝트 → `references/template-en.md`
- 신호가 서로 엇갈리거나 하나도 없으면 영문 템플릿을 쓴다. 외부 기여자가 읽을 가능성을 고려한 기본값이다. 어떤 신호로 골랐는지는 최종 보고에 한 줄로 남긴다.

선택한 템플릿 파일만 읽는다. 둘 다 읽을 필요는 없다.

## 5. PR 작성

- **제목**: 70자 이내 한 줄. 프로젝트가 Conventional Commits(`feat:`, `fix:` 등)를 쓰면 같은 형식으로, 아니면 변경을 설명하는 평서문으로 쓴다. 템플릿과 같은 언어로 쓴다.
- **본문**: 템플릿의 섹션을 모두 채운다. 해당 없는 섹션은 지우지 말고 "해당 없음"/"N/A"로 적는다. 구조가 일정해야 리뷰어가 빠르게 읽는다.
- 테스트를 실제로 돌리지 않았다면 돌렸다고 쓰지 않는다. 검증 항목은 실제로 확인한 것만 체크한다.
- 비밀정보·토큰·개인정보를 본문에 넣지 않는다.
- 이슈 번호가 브랜치명이나 커밋에 보이면 `Closes #N`을 연결한다. 없으면 만들어 내지 않는다.
- 본문 맨 끝에 다음 줄을 붙인다.
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`

## 6. push와 PR 생성

1. 현재 브랜치에 upstream이 없거나 원격보다 앞서 있으면 `git push -u origin <branch>`로 올린다. 사용자가 PR 생성을 요청한 것이 곧 이 push의 승인이다. force push는 하지 않는다.
2. 본문은 임시 파일이나 heredoc으로 `gh pr create --base <base> --title "<제목>" --body-file <파일>`에 전달한다. 따옴표·줄바꿈이 깨지지 않게 하기 위해서다. `--draft` 인자가 있으면 `--draft`를 붙인다.
3. 같은 브랜치의 PR이 이미 열려 있으면(`gh pr view`로 확인) 새로 만들지 말고 URL을 알리고 본문 갱신 여부만 보고한다.

## 7. 최종 보고

서브 에이전트의 결과만 메인 대화에 전달되므로, 아래를 간결하게 보고한다.

- PR URL
- base ← head 브랜치
- 사용한 템플릿(저장소 자체 / 영문 / 한국어)과 그렇게 고른 근거 한 줄
- 포함된 커밋 수와 변경 요지
- 주의사항(미커밋 변경, 테스트 미실행 등)
