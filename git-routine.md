---
description: 로컬 저장소 일괄 git 루틴 — vault 의 repos.md 권위 원본 기반. fetch → pull (clean) → dirty 일괄 commit+push
---

# /git-routine

Dr. Ben 의 매일 로컬 git 저장소 동기화 루틴.

## 권위 원본

대상 저장소 목록은 **vault 안 인벤토리** 가 단일 권위 원본:

```
~/projects/2nd-brain-vault/knowledge/02_areas/brain-system/repos.md
```

그 파일의 **"/git-routine 대상 (in-routine: yes)"** 표를 읽어 *경로 + 원격 + 카테고리* 를 추출한다. 어떤 repo 를 routine 대상으로 할지의 결정은 이 명령 파일이 아니라 위 인벤토리가 한다.

> 인벤토리에 항목 추가/제거가 *우선*. 이 명령 파일은 변경하지 않아도 됨.

## 인자

- `(없음)` 또는 `sync` — 전체 (fetch → pull → dirty 일괄 commit+push)
- `pull` — fetch + pull 만 (dirty 는 보고만, commit 안 함)
- `status` — read-only 점검 (어떤 저장소도 수정 안 함)

## 절차

### 0. 인벤토리 로드

`Read` 도구로 `~/projects/2nd-brain-vault/knowledge/02_areas/brain-system/repos.md` 읽기. "/git-routine 대상" 표에서 경로 목록 추출.

**폴백**: 인벤토리 파일이 없거나 읽기 실패 시:
- 사용자에게 경고하고, scan-only 모드로 전환 — `~/projects/*/.git`·`~/.claude/*/.git`·`~/.openclaw/workspace/.git` 자동 탐색.
- 폴백 모드에서는 `pull` 까지만 허용, `sync` (push 포함) 는 거부 (인벤토리 없이 push 는 위험).

### 1. 발견 + 상태 조사

각 인벤토리 항목에 대해 **병렬로**:

```bash
git -C <repo> fetch --quiet
git -C <repo> status --porcelain
git -C <repo> rev-list --left-right --count HEAD...@{u}   # ahead/behind
git -C <repo> rev-parse --abbrev-ref HEAD                 # 현재 브랜치
```

표 형식으로 보고:

| 카테고리 | 경로 | 브랜치 | ahead/behind | dirty 파일 수 |

특수 상태는 별도 줄: upstream 없는 브랜치·detached HEAD·`main` 이외의 브랜치.

### 1.5. Sanity check (자동 탐색)

별도로 `~/projects/*/.git`·`~/.claude/*/.git`·`~/.openclaw/workspace/.git` 스캔. 결과를 인벤토리와 비교:

- **인벤토리에 있는데 디스크에 없음** → "missing" 경고 + 이 PC 에서 clone 필요 여부 확인.
- **디스크에 있는데 인벤토리에 없음** → "untracked repo" 경고 + 인벤토리 갱신 또는 명시적 제외 사유 추가 권장.
- 인벤토리에 적힌 `/git-routine 제외` 목록과 일치하는 디스크 항목은 조용히 skip.

이 sanity check 결과는 *보고만* 함 — 자동으로 인벤토리·디스크를 고치지 않는다 (Dr. Ben 의 의사결정 필요).

### 2. Clean + behind 저장소: 자동 pull (`--ff-only`)

- dirty 한 곳은 건드리지 않음.
- non-FF 인 곳은 보고만 (자동 처리 안 함 — 사용자 판단 필요).
- 병렬 실행 가능.

### 3. Dirty 저장소 일괄 commit+push (`sync` 인자일 때만)

a. 모든 dirty 저장소의 변경 요약을 한 화면에 표시:
   - `git diff --stat` (수정·스테이지된 파일)
   - untracked 파일 목록
   - 각 저장소별 제안 commit 메시지 (변경 내용 기반)

b. **1회 일괄 확인** — "이대로 commit + push 해도 될까요?" Dr. Ben 승인 후 진행.

c. 저장소별로 순차 실행:
   - 변경된 파일만 **명시적으로** `git add <file> ...` (절대 `git add .` 또는 `-A` 금지)
   - `git commit -m "<제안 메시지>"`
   - `git push`
   - 실패 시 즉시 중단하고 어디서 멈췄는지 보고.

### 3.5. MCP user-scope 자동 적용 (`pull`·`sync` 둘 다)

`~/.claude/skills/` repo pull 이 완료된 직후, **`_mcp-servers/apply.sh`** 가 존재하면 자동 실행 — registry.json 의 user-scope MCP 정의를 `~/.claude.json` 에 멱등 반영. (skill 의 *디스크 파일 = 자동 등록* 모델을 MCP 의 *키 writeback* 으로 한 단계 매개.)

```bash
if [ -x "$HOME/.claude/skills/_mcp-servers/apply.sh" ]; then
  bash "$HOME/.claude/skills/_mcp-servers/apply.sh"
fi
```

- **멱등**: `= unchanged` 면 no-op, `↻ updated` 면 remove+re-add, `✓ added` 면 신규
- **orphan 경고만** — registry 에 없는 user-scope 서버는 자동 제거 X (수동 권장)
- `claude.ai *` connector 는 자동 skip (Anthropic OAuth 메커니즘)
- prereq (uvx·cmd.exe 등) 부재 시 `claude mcp list` 가 spawn 실패 표시 → 해당 머신에만 설치

상세 → `~/.claude/skills/_mcp-servers/README.md`.

### 4. 최종 요약

저장소별: pull 결과 / commit·push 결과 / skip 사유 / sanity check 경고 / **MCP apply 결과** (+M added, ↻N updated 등).

## 안전 규칙

- `git pull` 은 항상 `--ff-only` (non-FF 면 중단·보고).
- `git push --force` 금지.
- `git add .` / `git add -A` 금지 — `.env`·credentials 우발 포함 방지.
- `--no-verify` 등 hook 우회 금지.
- 새 commit 만 생성, `--amend` 금지.
- detached HEAD / upstream 없는 브랜치 / `main` 이외의 브랜치는 자동 commit·push 대상에서 제외하고 보고만.
- commit 메시지는 변경 내용 기반으로 제안하되 Co-Authored-By 트레일러는 붙이지 않음 (개인 vault·설정 동기 루틴이므로).
- 인벤토리(`repos.md`) 가 없으면 `sync` 거부 (위 폴백 참조). `pull`·`status` 만 허용.

## 참고

- 이 명령 자체가 `~/.claude/commands/` (= `BenKorea/claude-code-commands` 저장소) 안에 있어 본 명령의 sync 대상에도 포함된다 — 본 명령으로 본 명령을 푸시하는 자가-적용 가능.
- 새 저장소를 routine 에 추가할 때: 인벤토리 (`repos.md`) 표만 갱신하면 이 명령 파일은 손대지 않아도 됨.
- 일반화된 카테고리 정의·신규 사용자 onboarding 흐름은 vault-guide 의 `methodology/setup/prerequisite-repos.md` 참조.
