---
description: 현재 PC 의 자동발화 일괄 on/off/status — OpenClaw cron(docker 게이트웨이) + 모든 호스트 systemd 타이머(parser-drain·brain-drain·브라우저 사이드카) 통합 토글. "한 번에 한 PC 만 발화" 강제.
---

# /cron

현재 PC 의 **모든 자동발화**를 한 명령으로 일괄 토글. brain-system 자동화가 2곳으로 갈려 있어 통합 제어:

- **OpenClaw cron** — docker 게이트웨이(`2nd-brain-openclaw-gateway`) 안의 텍스트형 스킬 잡 (예: `gmail-label-actions-poll`)
- **systemd-user 타이머** — 게이트웨이 밖 호스트 자동발화 전부: (1) `parser-drain.timer`(extract — 듀얼 파싱), (2) `brain-drain.timer`(refine+brainify 무인 `claude -p`, 비용 발생), (3) 브라우저형 사이드카(예: `openclaw-webmail-sidecar.timer` → webmail-watch; 게이트웨이엔 브라우저가 없어 호스트 타이머로 분리).

다중 PC 에서 양쪽이 동시에 같은 자동발화를 돌리면 충돌(Telegram 중복 알림·Gmail 중복 forward·**parser-drain 이중 파싱 + SyncThing _parse 충돌**·**brain-drain 중복 노트·claude 비용 2배·파일 이동 경합**·사이트 polling 경합) → **한 번에 한 PC 만 활성**. 그래서 parser-drain 도 "결정형이라 상시" 가 아니라 *발화 머신에서만* 돈다(나머지 PC 는 수동 on-demand 만). 그 단일 토글을 명령 한 줄로.

## 인자

- `(없음)` 또는 `status` — 현재 PC 의 OpenClaw cron 잡 + 모든 호스트 타이머 상태 표시
- `off` — 활성 cron 일괄 disable + 모든 호스트 타이머 stop·disable
- `on` — 비활성 cron 일괄 enable + 모든 호스트 타이머 enable·start

## 제어 대상 2종

### A. OpenClaw cron (docker 게이트웨이)

- 게이트웨이 컨테이너 탐색: `GW=$(docker ps --filter name=openclaw-gateway --format '{{.Names}}' | head -1)`
- gateway 토큰: `TOK=$(docker exec "$GW" python3 -c "import json;print(json.load(open('/home/node/.openclaw/openclaw.json'))['gateway']['auth']['token'])")`
- **조회(빠름)**: 호스트 `~/.openclaw/cron/jobs.json` 직접 읽기 (`name`·`id`·`enabled`). CLI `cron list` 는 게이트웨이 왕복 ~10초라 read 에 부적합 (2026-05-17 실측). jobs.json 이 enabled 상태의 원본.
- **변경**: `docker exec "$GW" node /app/dist/index.js cron <enable|disable> <id> --token "$TOK"` (데몬 정합 보장하는 정식 경로 — jobs.json 수동편집 금지).
- 게이트웨이가 안 떠 있으면(`$GW` 비어 있음) OpenClaw cron 토글 불가 → 그 부분만 건너뛰고 보고.

### B. systemd 타이머 (호스트 자동발화 전부 — parser-drain·brain-drain·사이드카)

- 대상 탐색: `{ systemctl --user list-unit-files 'parser-drain.timer' 'brain-drain.timer' 'openclaw-*sidecar*.timer' --no-legend; } | awk '{print $1}'` (현재 = `parser-drain.timer` + `brain-drain.timer` + `openclaw-webmail-sidecar.timer`). 게이트웨이 밖 호스트 자동발화는 **전부** 여기로 — 단일 발화 머신 원칙(이중 파싱·중복 노트·_parse 충돌 방지).
- 조회: `systemctl --user is-active <timer>` · `is-enabled <timer>` · 다음 발화는 `systemctl --user list-timers <timer> --no-pager`
- **off**: `systemctl --user disable --now <timer>` (부팅 자동 해제 + 즉시 정지)
- **on**: `systemctl --user enable --now <timer>` (부팅 자동 + 즉시 활성)

### C. 호스트별 상태 마커 (SyncThing 동기 — 다른 PC 가시성)

- 위치: `~/projects/2nd-brain-vault/knowledge/02_areas/brain-system/cron-active-host.md` (vault 라 SyncThing 으로 양 PC 에 비춰짐).
- frontmatter 에 **PC 마다 두 줄**: `<hostname>_state: on|off` + `<hostname>_updated: <ISO8601>`. 항목 없으면 그 PC 는 `unknown`(아직 /cron 미기록). 머신명 하드코딩 없음 — 새 PC 는 자기 `<hostname>_*` 추가.
- **소유권 — 각 PC 는 자기 `${HN}_*` 두 줄만 쓴다**(`HN=$(hostname)`). 남의 항목은 *절대* 안 건드림 → 쓰기 주체가 안 겹쳐 SyncThing 충돌 0, 양쪽 동시 사고도 둘 다 `on` 으로 드러남.
- **읽기**: 다른 PC 의 마지막 cron 상태를 *물어보지 않고* 안다(이 명령이 PC 간 cron 상태를 묻던 문제 해소).
- ⚠️ **enabled 토글 자체는 동기 ✗**(per-machine — 동기하면 토글이 전파돼 단일발화가 깨짐). 이 마커는 *상태 가시성만* 위한 읽기전용 advisory. ground truth = 각 PC 실제 토글, 마커는 *마지막 /cron 액션* 기록. /cron 안 거치고 토글하면 어긋남(status 가 불일치 플래그).
- 편집은 frontmatter 의 자기 두 줄만 교체(없으면 추가). 파싱: `grep '^<host>_state:'` 식 단순 key:value.

## 절차

### status

1. **hostname 출력** (어느 PC 인지 또렷이).
2. **A — OpenClaw cron**: `~/.openclaw/cron/jobs.json` 읽어 표 (`이름 | 상태 | ID`). 게이트웨이 미실행이면 "게이트웨이 미실행 — cron 토글 불가" 표시(jobs.json 은 그래도 읽힘).
3. **B — systemd 타이머**: 호스트 타이머별(parser-drain·brain-drain·사이드카) `active`·`enabled`·다음 발화 표. (NEXT 가 비어 있으면 재무장 결함 신호 — `OnActiveSec` 시드 확인. 단 list-timers NEXT 가 "-"라도 `systemctl status` 의 `Trigger:` 가 잡혀 있으면 정상 — 시계 skew 표시 quirk.)
4. **C — 호스트별 마커 읽기 (§C)**: `cron-active-host.md` 의 **모든** `*_state` 항목 보고 — `kimbi: on (10:07)` · `ai4lt: unknown` 식으로 각 PC 상태+시각. **이 PC 항목을 실제 상태와 대조**: 마커=on 인데 실제 off(or 반대)면 **불일치 경고**(/cron 안 거친 토글 or stale). → 이로써 *다른 PC cron 상태를 물어볼 필요가 없다*.
5. **요약 줄**: `<hostname> — cron enabled N/disabled M · 타이머 active K/total T · 마커: 이PC=on, 타PC=<상태>`.

### off

1. **A**: jobs.json 의 `enabled=true` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron disable <id> --token "$TOK"` 순차. 실패 시 즉시 중단·보고.
2. **B**: 활성 호스트 타이머 전부 → `systemctl --user disable --now <timer>`.
3. **C — 마커 갱신 (§C)**: 이 PC 의 `${HN}_state: off` + `${HN}_updated: <now>` 로 *자기 두 줄만* 갱신(다른 PC 항목 불변).
4. 결과 표 + 요약 ("이 PC: cron N개 disable + 타이머 M개 정지 · 마커 ${HN}→off").
5. 이미 모두 정지면 "이 PC 엔 활성 자동발화 없음" 보고(마커 `${HN}_state` 도 off 로 정리).

### on

0. **C — 마커 선확인 (§C, 이중발화 가드)**: `cron-active-host.md` 에서 *다른* PC 의 `<host>_state` 가 `on` 이면 → **이중발화 경고**: "마커상 `<host>` 가 발화 중입니다(updated …). 그 PC 를 먼저 `/cron off` 안 하면 충돌(이중 파싱·중복 노트·Contact 중복). 그래도 이 PC 를 켤까요?" Dr. Ben 확인 후 진행. (다른 PC 가 off·unknown 이면 조용히 진행.)
1. **A**: jobs.json 의 `enabled=false` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron enable <id> --token "$TOK"` 순차.
2. **B**: 호스트 타이머 전부 → `systemctl --user enable --now <timer>` (`--now` 가 OnActiveSec 시드 발화 → 체인 시작).
3. **C — 마커 갱신 (§C)**: 이 PC 의 `${HN}_state: on` + `${HN}_updated: <now>` 로 *자기 두 줄만* 갱신(없으면 추가).
4. 결과 표 + 요약 ("이 PC: cron N개 enable + 타이머 M개 활성 · 마커 ${HN}→on").

## 안전 규칙

- 이 명령은 **현재 PC** 만 토글 — cron `enabled`·타이머 enable 둘 다 **머신별 로컬**(동기 대상 아님). 다른 PC 상태는 **§C 호스트별 상태 마커(SyncThing)로 *읽어* 확인**(토글은 여전히 현재 PC 만, 마커는 가시성 전용 — 동기되는 건 *상태 기록*이지 *토글*이 아니다). 마커 쓰기는 **자기 `<hostname>_*` 항목만** — 남의 항목 절대 안 건드림.
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
- **parser-drain 타이머 정의** = `~/.config/systemd/user/parser-drain.{service,timer}` (PC별). 본체·설치는 `2nd-brain/docker/parser-drain/` (`cd ~/projects/2nd-brain/docker && make install-parser-drain`). extract(듀얼 파싱).
- **brain-drain 타이머 정의** = `~/.config/systemd/user/brain-drain.{service,timer}` (PC별). 본체·설치는 `2nd-brain/automation/brain-drain/` (`make install-brain-drain`). refine+brainify 무인 드레인.
- **사이드카 타이머 정의** = `~/.config/systemd/user/openclaw-*sidecar*.{service,timer}` (PC별). 사이드카 compose·이미지는 `2nd-brain/docker/webmail-sidecar/`.
- 실제 활성 상태(cron `enabled` / 타이머 enable)는 **PC 마다 독립** — git/SyncThing 동기 대상 아님.
- **호스트별 상태 마커** = `~/projects/2nd-brain-vault/knowledge/02_areas/brain-system/cron-active-host.md` (vault, SyncThing 동기). PC 마다 `<hostname>_state`/`_updated` 두 줄, **각 PC 가 자기 것만 기록·남은 읽기**. enabled 토글은 미동기지만 *각 PC 가 발화 중인지* 가시성은 이 advisory 마커로 공유 — /cron on/off 가 자기 항목 갱신(§C), status 가 전부 읽음. (2026-05-27 신설: PC 간 cron 상태를 매번 물어보던 문제 해소. 초기 단일 `active_host` 포인터 → Dr. Ben 검토로 PC별 항목으로 개정 — 각자 자기 것만 써 충돌·덮어쓰기 0.)
- 타이머는 모두 `OnActiveSec`(시드) + `OnUnitInactiveSec`(no-overlap 재무장) 패턴 — `enable --now`/재로드마다 재무장(인라인 주석 금지: bad-setting 으로 조용히 폐기됨, 2026-05-26 결함 교훈).
- 향후 사이드카 추가(society-watch 등) 시 `openclaw-*sidecar*.timer` 패턴이 자동 포함. parser-drain·brain-drain 외 새 호스트 드레인 추가 시 탐색 목록에 명시 추가.
