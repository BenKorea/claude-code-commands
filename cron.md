---
description: 현재 PC 의 자동발화 일괄 on/off/status — OpenClaw cron(docker 게이트웨이) + systemd 타이머(브라우저 사이드카) 통합 토글
---

# /cron

현재 PC 의 **모든 자동발화**를 한 명령으로 일괄 토글. brain-system 자동화가 2곳으로 갈려 있어 통합 제어:

- **OpenClaw cron** — docker 게이트웨이(`2nd-brain-openclaw-gateway`) 안의 텍스트형 스킬 잡 (예: `gmail-label-actions-poll`)
- **systemd-user 타이머** — 호스트의 브라우저형 사이드카 (예: `openclaw-webmail-sidecar.timer` → webmail-watch). 게이트웨이엔 브라우저가 없어 cron 으로 못 돌리므로 호스트 타이머로 분리됨.

다중 PC 에서 양쪽이 동시에 같은 자동발화를 돌리면 충돌(Telegram 중복 알림·Gmail 중복 forward·state 어긋남·사이트 polling 경합) → **한 번에 한 PC 만 활성** 운영 권장. 그 토글을 명령 한 줄로.

## 인자

- `(없음)` 또는 `status` — 현재 PC 의 OpenClaw cron 잡 + systemd 사이드카 타이머 상태 표시
- `off` — 활성 cron 일괄 disable + 사이드카 타이머 stop·disable
- `on` — 비활성 cron 일괄 enable + 사이드카 타이머 enable·start

## 제어 대상 2종

### A. OpenClaw cron (docker 게이트웨이)

- 게이트웨이 컨테이너 탐색: `GW=$(docker ps --filter name=openclaw-gateway --format '{{.Names}}' | head -1)`
- gateway 토큰: `TOK=$(docker exec "$GW" python3 -c "import json;print(json.load(open('/home/node/.openclaw/openclaw.json'))['gateway']['auth']['token'])")`
- **조회(빠름)**: 호스트 `~/.openclaw/cron/jobs.json` 직접 읽기 (`name`·`id`·`enabled`). CLI `cron list` 는 게이트웨이 왕복 ~10초라 read 에 부적합 (2026-05-17 실측). jobs.json 이 enabled 상태의 원본.
- **변경**: `docker exec "$GW" node /app/dist/index.js cron <enable|disable> <id> --token "$TOK"` (데몬 정합 보장하는 정식 경로 — jobs.json 수동편집 금지).
- 게이트웨이가 안 떠 있으면(`$GW` 비어 있음) OpenClaw cron 토글 불가 → 그 부분만 건너뛰고 보고.

### B. systemd 타이머 (호스트 브라우저 사이드카)

- 대상 탐색: `systemctl --user list-unit-files 'openclaw-*sidecar*.timer' --no-legend | awk '{print $1}'` (현재 = `openclaw-webmail-sidecar.timer`)
- 조회: `systemctl --user is-active <timer>` · `is-enabled <timer>` · 다음 발화는 `systemctl --user list-timers <timer> --no-pager`
- **off**: `systemctl --user disable --now <timer>` (부팅 자동 해제 + 즉시 정지)
- **on**: `systemctl --user enable --now <timer>` (부팅 자동 + 즉시 활성)

## 절차

### status

1. **hostname 출력** (어느 PC 인지 또렷이).
2. **A — OpenClaw cron**: `~/.openclaw/cron/jobs.json` 읽어 표 (`이름 | 상태 | ID`). 게이트웨이 미실행이면 "게이트웨이 미실행 — cron 토글 불가" 표시(jobs.json 은 그래도 읽힘).
3. **B — systemd 타이머**: 사이드카 타이머별 `active`·`enabled`·다음 발화 표.
4. **요약 줄**: `<hostname> — cron enabled N/disabled M · 타이머 active K/total T`.

### off

1. **A**: jobs.json 의 `enabled=true` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron disable <id> --token "$TOK"` 순차. 실패 시 즉시 중단·보고.
2. **B**: 활성 사이드카 타이머 → `systemctl --user disable --now <timer>`.
3. 결과 표 + 요약 ("이 PC: cron N개 disable + 타이머 M개 정지").
4. 이미 모두 정지면 "이 PC 엔 활성 자동발화 없음" 보고.

### on

1. **A**: jobs.json 의 `enabled=false` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron enable <id> --token "$TOK"` 순차.
2. **B**: 사이드카 타이머 → `systemctl --user enable --now <timer>`.
3. 결과 표 + 요약.

## 안전 규칙

- 이 명령은 **현재 PC** 만 토글 — cron `enabled`·타이머 enable 둘 다 **머신별 로컬**(동기 대상 아님). 다른 PC 상태는 모름.
- **조회=직접읽기**(jobs.json·systemctl is-active), **변경만 정식경로**(docker exec cron / systemctl). jobs.json 의 `enabled` 를 수동편집하지 말 것 — `cron <verb>` CLI 가 데몬 정합 보장.
- 양쪽 PC 가 *모두 off* 로 잊히면 아무 데서도 안 돈다 → status 요약에 hostname 또렷이 표시.
- 게이트웨이 미실행이면 A(cron) 토글 불가 — B(타이머)만 처리하고 그 사실 보고.
- (native 운영 PC: docker 게이트웨이가 없으면 A 는 호스트 `openclaw cron <verb> <id>` 가 정식 경로. Dr. Ben 현 환경은 **docker-only** 라 `docker exec` 우선 — 2026-05-25 docker cutover.)

## 권장 운영 워크플로우 (PC 이동)

1. **떠나는 PC**: `/cron off` — cron + 타이머 둘 다 정지.
2. **떠나는 PC**: 변경 있으면 `/git-routine sync` 로 push.
3. **이동.**
4. **새 PC**: `/git-routine pull`.
5. **새 PC**: `/cron on` — cron + 타이머 활성화.

→ (a) 양쪽 동시 발화 없음, (b) 새 PC 가 옛 코드로 안 돎.

## 참고

- **OpenClaw cron 정의 원본** = `~/.openclaw/cron/jobs.json` (PC별·미동기). 게이트웨이가 mount 해서 읽음. (구 `openclaw-config/cron/jobs.json.template` 는 2026-05-25 openclaw-config 은퇴로 폐기 — 이제 jobs.json 이 원본.)
- **사이드카 타이머 정의** = `~/.config/systemd/user/openclaw-*sidecar*.{service,timer}` (PC별). 사이드카 compose·이미지는 `2nd-brain/docker/webmail-sidecar/`.
- 실제 활성 상태(cron `enabled` / 타이머 enable)는 **PC 마다 독립** — git/SyncThing 동기 대상 아님.
- 향후 사이드카 추가(society-watch 등) 시 `openclaw-*sidecar*.timer` 패턴이 자동 포함.
