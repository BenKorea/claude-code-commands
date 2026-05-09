# claude-code-commands

Dr. Ben의 Claude Code (WSL2 CLI) 커스텀 슬래시 커맨드.

## 배치 규칙

각 커맨드는 단일 마크다운 파일:
```
commands/
├── brainify.md       /brainify — 외부 정보 브레인화
├── gmail-batch.md    /gmail-batch — 인박스 일괄 브레인화
└── (추가 커맨드)
```

파일명이 슬래시 커맨드 이름이 됨 (`brainify.md` → `/brainify`). Claude Code (`~/.claude/commands/`)가 이 폴더를 직접 읽음.

## 새 머신 onboarding (WSL2)

기존에 `~/.claude/commands` 가 Google Drive symlink 였다면 끊고 git clone 으로 대체:

```bash
# 1. 기존 symlink 제거
rm ~/.claude/commands

# 2. 이 repo 를 직접 clone
git clone https://github.com/BenKorea/claude-code-commands.git ~/.claude/commands
```

검증:
```bash
ls -la ~/.claude/commands         # symlink 가 아닌 디렉토리여야 함
git -C ~/.claude/commands status  # repo 인식 확인
```

## 자매 repo

- [claude-code-skills](https://github.com/BenKorea/claude-code-skills) — Claude Code 스킬 정의. 동일 onboarding 절차 + 관련 자산 안내 포함.
