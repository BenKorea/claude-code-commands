---
description: OpenClaw cron 일괄 on/off/status — 다중 PC 동시 발화 방지용 머신별 토글
---

# /cron

OpenClaw cron job 을 현재 PC 에서 일괄 토글. 노트북·데스크탑 같은 PC 에서 양쪽 동시에 같은 cron 이 돌면 충돌 (Telegram 중복 알림·state 어긋남·외부 사이트 polling 충돌 등) 이 발생하므로, **한 번에 한 PC 만 활성** 운영이 권장. 그 토글을 명령 한 줄로.

## 인자

- `(없음)` 또는 `status` — 현재 PC 의 모든 cron job (활성·비활성 모두) 표시
- `off` — 모든 *활성* cron 을 일괄 disable
- `on` — 모든 *비활성* cron 을 일괄 enable

## 절차

### status

먼저 hostname 출력 (어느 PC 인지 또렷이 보기 위함). 그 다음 **`~/.openclaw/cron/jobs.json` 을 직접 읽어** (python `json` 등) `name`·`id`·`enabled` 추출, 표로 보고:

| 이름 | 상태 | ID |

요약 줄에 hostname 과 카운트: `<hostname> — enabled: N / disabled: M / total: K`.

> **왜 `openclaw cron list` 가 아니라 jobs.json 직접 읽기인가** (2026-05-17 실측): `openclaw cron list --all --json` 은 실행 중인 OpenClaw 게이트웨이/런타임과 왕복해 **~10초** 걸린다 (`openclaw --version` 0.18초 — 일반 Node 부팅 아님, 이 서브커맨드 고유 비용). jobs.json 은 enabled 상태의 *원본* 이고 직접 읽으면 **~0.04초** (250배). status 는 read-only 라 게이트웨이 왕복 불필요 — `enabled`·`name`·`id` 만 필요하고 그건 전부 jobs.json 에 있다. (CLI 가 더 주는 라이브 `nextRun` 은 실측상 `None` 으로 안 나옴 → "다음 발화" 컬럼 폐지, 손실 0.) **조회는 jobs.json 직접 / 변경(enable·disable)만 `openclaw cron <verb> <id>` CLI** — 이 분리가 핵심.

### off

1. `~/.openclaw/cron/jobs.json` 직접 읽어 `enabled=true` 만 필터링 (열거=조회라 CLI 불필요).
2. 각 ID 에 대해 `openclaw cron disable <id>` 순차 실행 (변경은 CLI 정식 경로). 실패 시 즉시 중단·보고.
3. 결과 표 + 요약 ("이 PC 에서 N개 disable 완료").
4. 이미 모두 disabled 인 경우 "이 PC 에는 활성 cron 이 없습니다" 만 보고하고 종료.

### on

1. `~/.openclaw/cron/jobs.json` 직접 읽어 `enabled=false` 만 필터링 (열거=조회라 CLI 불필요).
2. 각 ID 에 대해 `openclaw cron enable <id>` 순차 실행 (변경은 CLI 정식 경로). 실패 시 즉시 중단·보고.
3. 결과 표 + 요약 ("이 PC 에서 N개 enable 완료").
4. 이미 모두 enabled 인 경우 "이 PC 에는 비활성 cron 이 없습니다" 만 보고하고 종료.

## 안전 규칙

- 이 명령은 **현재 PC** 의 cron 만 토글한다 — 다른 PC 상태는 알 수 없음.
- 양쪽 PC 가 *모두 off* 인 상태로 잊혀지면 cron 이 아무 데서도 안 돈다. status 요약 줄에 hostname 을 또렷이 표시해 어느 PC 를 보고 있는지 즉시 인지하도록 한다.
- **열거·조회는 `jobs.json` 직접 읽기, 변경(enable/disable)만 `openclaw cron` CLI.** CLI list 는 게이트웨이 왕복 ~10초라 read 에 부적합 (status 절차 주석 참조). jobs.json 의 `enabled` 가 *이 PC* 현재 상태의 원본 (PC 별 독립·동기 대상 아님 — 아래 참고 섹션).
- cron job 정의 자체 (`~/.openclaw/cron/jobs.json` 의 schedule·message 등) 는 건드리지 않고, 변경 시 `openclaw cron <verb>` 가 *enabled 필드만* 토글한다 (데몬 정합 보장하는 정식 경로 — 수동 jobs.json 편집으로 enabled 바꾸지 말 것).

## 권장 운영 워크플로우

PC 간 이동 시:

1. **떠나는 PC**: `/cron off` — 활성 cron 모두 정지.
2. **떠나는 PC**: 작업 중이던 변경이 있다면 `/git-routine sync` 로 push.
3. **이동.**
4. **새 PC**: `/git-routine pull` 로 다른 PC 가 push 한 변경 받기.
5. **새 PC**: `/cron on` — cron 활성화.

이 순서를 지키면 (a) 양쪽 동시 발화 없음, (b) 새 PC 가 옛 코드로 cron 을 도는 일 없음.

## 참고

- cron 정의의 *재현 가능한 골격* 은 `openclaw-config/cron/jobs.json.template` (`BenKorea/openclaw-config` repo). 새 PC 셋업 시 `setup.sh` 가 chat_id 채워 `~/.openclaw/cron/jobs.json` 으로 배치.
- 실제 활성·비활성 상태 (jobs.json 의 `enabled` 필드) 는 PC 마다 *독립적* — git/SyncThing 동기 대상이 아님.
- v1 은 사용자가 PC 별 토글 상태를 *기억해야 한다*. 향후 사고가 반복되면 (a) vault 안 SyncThing 공유 마커 파일에 "현 활성 호스트" 기록, 또는 (b) Telegram bot 으로 off/on announce 같은 안전장치 추가 검토.
