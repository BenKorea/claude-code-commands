---
description: 현재 PC 의 자동발화 일괄 on/off/status — OpenClaw cron(docker 게이트웨이) + 모든 호스트 systemd 타이머(parser-drain·brain-drain·브라우저 사이드카) 통합 토글. "한 번에 한 PC 만 발화" 강제.
---

# /cron

현재 PC 의 **모든 자동발화**를 한 명령으로 일괄 토글. brain-system 자동화가 2곳으로 갈려 있어 통합 제어:

- **OpenClaw cron** — docker 게이트웨이(`2nd-brain-openclaw-gateway`) 안의 텍스트형 스킬 잡 (예: `gmail-label-actions-poll`)
- **systemd-user 타이머** — 게이트웨이 밖 호스트 자동발화 전부: (1) `parser-drain.timer`(extract — 듀얼 파싱), (2) `brain-drain.timer`(refine+brainify 무인 `claude -p`, 비용 발생), (3) 브라우저형 사이드카(예: `openclaw-webmail-sidecar.timer` → webmail-watch; 게이트웨이엔 브라우저가 없어 호스트 타이머로 분리), (4) `brain-health.timer`(아침 헬스체크·텔레그램 보고 — 읽기 전용 관측).

다중 PC 에서 양쪽이 동시에 같은 자동발화를 돌리면 충돌(Telegram 중복 알림·Gmail 중복 forward·**parser-drain 이중 파싱 + SyncThing _parse 충돌**·**brain-drain 중복 노트·claude 비용 2배·파일 이동 경합**·사이트 polling 경합) → **한 번에 한 PC 만 활성**. 그래서 parser-drain 도 "결정형이라 상시" 가 아니라 *발화 머신에서만* 돈다(나머지 PC 는 수동 on-demand 만). 그 단일 토글을 명령 한 줄로.

## 인자

- `(없음)` 또는 `status` — 현재 PC 의 OpenClaw cron 잡 + 모든 호스트 타이머 상태 표시
- `off` — 활성 cron 일괄 disable + 모든 호스트 타이머 stop·disable
- `on` — 비활성 cron 일괄 enable + 모든 호스트 타이머 enable·start

## 제어 대상 2종

### A. OpenClaw cron (docker 게이트웨이)

- 게이트웨이 컨테이너 탐색: `GW=$(docker ps --filter name=openclaw-gateway --format '{{.Names}}' | head -1)`
- gateway 토큰: `TOK=$(docker exec "$GW" python3 -c "import json;print(json.load(open('/home/node/.openclaw/openclaw.json'))['gateway']['auth']['token'])")`
- **조회 (권위 = SQLite)**: ⚠️ 2026.6.8 부터 cron 스토어가 평면 JSON(`~/.openclaw/cron/jobs.json`) → **SQLite(`~/.openclaw/state/openclaw.sqlite` 의 `cron_jobs` 테이블)** 로 이전됨. 구 `jobs.json` 은 `jobs.json.migrated` 로 개명돼 **더 이상 없음** — 절대 이걸 진실의 원천으로 읽지 말 것(읽으면 "잡 비어있음=유실" 오판 → 매번 중복 재등록되는 함정. 2026-06-24 규명). 권위 조회 = `docker exec "$GW" node /app/dist/index.js cron list --all --json --token "$TOK"` (게이트웨이 왕복 ~10초지만 유일 권위. `--all` 없으면 disabled 잡이 안 보임). 더 빠른 직접 read 가 필요하면 호스트 SQLite 를 node:sqlite read-only 로: `docker exec "$GW" node -e '…SELECT job_id,name,enabled,schedule_expr FROM cron_jobs…'`(store_key 무관 전부 본다 — 옛 store_key 의 고아 잡까지 드러남). `~/.openclaw/state/` 는 bind-mount 라 영속(재시작·업그레이드 생존).
- **변경**: `docker exec "$GW" node /app/dist/index.js cron <enable|disable|rm> <id> --token "$TOK"` (데몬 정합 보장하는 정식 경로 — SQLite 직접편집 금지). 잡이 통째로 없으면(마이그레이션 유실 등) `cron add` 로 재등록하되 **먼저 `cron list --all` 로 중복 여부 확인** (중복이면 add 말고 enable).
- 게이트웨이가 안 떠 있으면(`$GW` 비어 있음) OpenClaw cron 토글 불가 → 그 부분만 건너뛰고 보고.

### B. systemd 타이머 (호스트 자동발화 전부 — parser-drain·brain-drain·사이드카)

- 대상 탐색: `{ systemctl --user list-unit-files 'parser-drain.timer' 'brain-drain.timer' 'brain-health.timer' 'openclaw-*sidecar*.timer' --no-legend; } | awk '{print $1}'` (현재 = `parser-drain.timer` + `brain-drain.timer` + `brain-health.timer` + `openclaw-webmail-sidecar.timer`). 게이트웨이 밖 호스트 자동발화는 **전부** 여기로 — 단일 발화 머신 원칙(이중 파싱·중복 노트·_parse 충돌 방지).
  - **`brain-health.timer` 도 같이 토글하는 이유** (2026-08-04 신설): 이 타이머는 쓰기 없는 *관측*이라 충돌 위험이 없지만, **발화 머신 = 감시 대상 = 보고 머신**이어야 한다. 꺼진 머신에서 계속 돌면 ① 아침 보고가 두 통 오고 ② 그 머신의 타이머가 전부 꺼져 있으니 "자동 작업 꺼짐" 을 문제로 오보고한다. 그래서 발화와 함께 켜고 끈다.
- 조회: `systemctl --user is-active <timer>` · `is-enabled <timer>` · 다음 발화는 `systemctl --user list-timers <timer> --no-pager`
- **off**: `systemctl --user disable --now <timer>` (부팅 자동 해제 + 즉시 정지)
- **on**: `systemctl --user enable --now <timer>` (부팅 자동 + 즉시 활성)

### C. cron 상태 마커 — Google Sheet (gog, 즉시 클라우드)

다른 PC 의 cron 상태를 *물어보지 않고* 알기 위한 advisory 마커. SyncThing 다단계 체인의 전파 지연·**잠들기 race**(전파 전 노트북 닫으면 유실)를 피해 **gog 로 Google Sheet 에 동기적 클라우드 read/write** — 쓰기가 리턴된 순간 이미 클라우드 영속이라 직후 잠들어도 안전(2026-05-27, 인터넷 상시연결 전제).

- **Sheet**: spreadsheetId `1eXlbYvKVtAo5GEKTBjFp9Uu3t1XDFxee4_vuQFxRfBw`, 탭 `cron-status`, 열 `A=host B=state C=updated`(1행 헤더, 호스트당 1행). 사람 보기: `https://docs.google.com/spreadsheets/d/1eXlbYvKVtAo5GEKTBjFp9Uu3t1XDFxee4_vuQFxRfBw/edit`
- **gog 전제**: 호스트 gog, 계정 `kimbi.kirams@gmail.com`, 비대화식이라 **`GOG_KEYRING_PASSWORD` 필요**. 매 gog 호출 전 export. **정본 = Bitwarden**(`gog keyring — <host> host`), **runtime cache = 머신로컬 파일 `~/.config/gogcli/.keyring-password`**(mode 600, 머신별 독립 — keyring 자체가 머신별 자물쇠라 *동기 금지*). 로드:
  ```bash
  if [ -r ~/.config/gogcli/.keyring-password ]; then
    export GOG_KEYRING_PASSWORD=$(cat ~/.config/gogcli/.keyring-password)
  fi
  ```
  파일 없으면 §C 단계만 advisory 실패(non-fatal — 토글은 진행). 비번 망실 시 = `gog auth add kimbi.kirams@gmail.com` 재인증 1회로 복구(데이터 손실 0). Bitwarden 갱신 시 이 파일도 함께 갱신.
- **읽기**: `gog sheets get <SID> 'A2:C50' -a kimbi.kirams@gmail.com -p` → TSV(`host⇥state⇥updated`) 행 파싱.
- **쓰기 (자기 행만)**: 읽은 행에서 `A`열==`$(hostname)` 인 행번호 N 찾기 → 있으면 `gog sheets update <SID> "A{N}:C{N}" "<host>|<state>|<ISO8601>"` (**파이프=셀, 쉼표=행**), 없으면 `gog sheets append <SID> 'A:C' "<host>|<state>|<ts>"`.
- **소유권**: 각 PC 는 *자기 host 행만* 갱신, 남의 행 불변 → 쓰기 주체 안 겹침, 양쪽 동시 사고도 둘 다 `on` 으로 드러남.
- **non-fatal**: gog 실패(인증·네트워크·keyring)면 마커 단계만 경고하고 **토글은 그대로 진행**(마커는 advisory, 토글이 우선). ground truth = 각 PC 실제 토글, Sheet 는 *마지막 /cron 액션* 기록.
- ⚠️ **enabled 토글 자체는 동기 ✗**(per-machine — 동기하면 토글 전파로 단일발화 깨짐). Sheet 는 *상태 가시성만*.

## 절차

### status

1. **hostname 출력** (어느 PC 인지 또렷이).
2. **게이트웨이 헬스 (A 의 전제 — 3시점 점검)**. 게이트웨이가 떠 있으면 아래 셋 다 확인, 미실행이면 이 단계 통째 건너뛰고 A 의 SQLite 직접 read 만 함(`cron list` 는 게이트웨이 필요 → 미실행 시 node:sqlite read-only 로 `cron_jobs` 조회).
   - **inside-out (자가진단·가장 깊음)**: `docker exec "$GW" node /app/dist/index.js doctor --lint --severity-min error --deep` → `ok:true` + `findings:[]` 이어야 *진짜 정상*. ⚠ 기본 `--lint` 의 `ok:false` 는 advisory warning(평문 토큰·`lan` 바인딩·미설치 스킬 default-allow 등 ~50건) — error 만 진짜 신호라 `--severity-min error` 가 권위. doctor 는 채널(Telegram 토큰·pairing)·모델·최근 세션·skills-readiness 까지 봐 healthz 만으론 못 잡는 회귀(봇 토큰 만료·모델 misconfig 등)를 잡음.
   - **outside-in (도달성·도커 상태)**: `docker ps --filter name=openclaw-gateway --format '{{.Status}}'` 가 `Up ... (healthy)`; `curl -fsS http://127.0.0.1:18789/healthz` 가 `{"ok":true,"status":"live"}`. doctor 는 컨테이너 *안쪽* 시점이라 호스트→컨테이너 포트 도달성·도커 헬스체크는 별도.
   - **회귀 가드 (PATH fix·번들 claude)**: `docker exec "$GW" claude --version` 이 `2.1.x (Claude Code)`. 비면 extra.yml 의 PATH 라인이 깨졌다는 신호 — 채널 메시지가 ENOENT/EPIPE 로 죽음(install doc §5 의 핵심 회귀).
   - **라이브 auth 프로브 (자격 만료·refresh 부재 — doctor 사각지대)**: ⚠️ doctor·`auth status` 는 *저장 파일 메타*(계정·구독·하니스)만 봐 **토큰이 실제로 살아있는지 라이브 검증을 안 한다** → 게이트웨이 OAuth 자격이 만료됐어도 `ok:true`/`loggedIn:true` 로 통과(2026-06-30 사고: `~/.openclaw/.claude/.credentials.json` 의 refreshToken EMPTY → access token 만료 후 자가갱신 불가 → cron 턴만 401, doctor green). 그래서 **1-토큰 라이브 호출로 ground-truth 확인**: `docker exec "$GW" sh -c "echo 'reply OK' | /home/node/.openclaw/bin/claude -p 2>&1 | head -5"`. 출력이 `OK` 면 auth 살아있음 ✅; `401`/`Invalid authentication`/`Failed to authenticate` 가 보이면 **auth DEAD** ❌. (bare `claude` 는 sh PATH 에 없음 → 절대경로 필수. `--model` 지정 말 것 — agent 라우팅 밖이라 모델에러로 오인됨, 기본 모델로.) 구조적 원인 동시확인 = `python3` 로 `~/.openclaw/.claude/.credentials.json` 의 `claudeAiOauth.{refreshToken,expiresAt}` (refreshToken 빈 문자열 또는 expiresAt 과거 = 죽었거나 곧 죽음). **수리**(브라우저 OAuth라 Dr. Ben 직접 `!`): `! CLAUDE_CONFIG_DIR=/home/ben/.openclaw/.claude claude auth login` — access+refresh 둘 다 발급(=자가갱신 복원). `setup-token` 은 refresh 없어 재발 ✗. → 메모리 `reference_openclaw_gateway_oauth_expired_no_refresh`.
3. **A — OpenClaw cron**: `cron list --all --json` (또는 SQLite `cron_jobs` 직접 read) 로 표 (`이름 | 상태 | ID`). 게이트웨이 미실행이면 "게이트웨이 미실행 — cron 토글 불가" 표시(SQLite 직접 read 는 그래도 가능). **중복 잡 가드**: 같은 name 이 2건 이상이거나 enabled 가 2건 이상이면 경고 — 마이그레이션 유실 후 중복 재등록의 흔적(2026-06-24). 정리는 disabled 중복을 `cron rm`.
4. **B — systemd 타이머**: 호스트 타이머별(parser-drain·brain-drain·사이드카) `active`·`enabled`·다음 발화 표. (NEXT 가 비어 있으면 재무장 결함 신호 — `OnActiveSec` 시드 확인. 단 list-timers NEXT 가 "-"라도 `systemctl status` 의 `Trigger:` 가 잡혀 있으면 정상 — 시계 skew 표시 quirk.)
5. **C — Sheet 마커 읽기 (§C)**: `gog sheets get` 로 **모든 행** 보고 — `kimbi: on (10:41)` · `ai4lt: off (…)` 식 각 PC 상태+시각. **이 PC 행을 실제 상태와 대조**: 마커=on 인데 실제 off(or 반대)면 **불일치 경고**(/cron 안 거친 토글 or stale). → 이로써 *다른 PC cron 상태를 물어볼 필요가 없다*. (gog 실패 시 "마커 읽기 실패(gog: …)" 만 보고하고 나머지 status 는 계속.)
6. **요약 줄**: `<hostname> — gateway <healthy|warn|down> · auth <ok|DEAD> · cron enabled N/disabled M · 타이머 active K/total T · 마커: 이PC=<상태>, 타PC=<상태>`. (gateway: 4시점 모두 통과=`healthy`, doctor warning 만=`healthy(warn)`, doctor error·outside-in 실패=`unhealthy`, 미실행=`down`. **auth 프로브 DEAD 는 doctor green 이어도 별도 줄로 크게 경고** + 위 수리명령 제시 — cron 을 켜도 전부 401 로 조용히 실패하므로 가장 치명적.)

### off

1. **A**: `cron list --all --json` 의 `enabled=true` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron disable <id> --token "$TOK"` 순차. 실패 시 즉시 중단·보고.
2. **B**: 활성 호스트 타이머 전부 → `systemctl --user disable --now <timer>`.
3. **C — Sheet 마커 갱신 (§C)**: 이 PC 의 host 행을 `<host>|off|<now>` 로 *자기 행만* 갱신(없으면 append). gog 실패 시 경고만 하고 토글 결과는 유효(non-fatal).
4. 결과 표 + 요약 ("이 PC: cron N개 disable + 타이머 M개 정지 · 마커 <host>→off").
5. 이미 모두 정지면 "이 PC 엔 활성 자동발화 없음" 보고(마커 행도 off 로 정리).

### on

0. **C — Sheet 선확인 (§C, 이중발화 가드)**: `gog sheets get` 으로 *다른* PC 행의 `state` 가 `on` 이면 → **이중발화 경고**: "마커상 `<host>` 가 발화 중입니다(updated …). 그 PC 를 먼저 `/cron off` 안 하면 충돌(이중 파싱·중복 노트·Contact 중복). 그래도 이 PC 를 켤까요?" Dr. Ben 확인 후 진행. (다른 PC 가 off·없음이면 조용히 진행. gog 실패 시 가드 불가 — 경고하고 사용자 판단으로 진행.)
0.5. **auth 라이브 게이트 (죽은 자격으로 켜기 방지 — 이게 `/cron on` 의 핵심 가드)**: A enable 전에 §status 2 의 *라이브 auth 프로브* 1회 실행. 출력에 `401`/`Invalid authentication`/`Failed to authenticate` 가 있으면 → **A(cron) enable 중단·경고**: "게이트웨이 OAuth 자격이 죽어 있어(원인 가능성: refreshToken 부재) cron 을 켜도 `gmail-label-actions-poll` 이 매번 401 로 *조용히* 전부 실패합니다. 먼저 재로그인하세요: `! CLAUDE_CONFIG_DIR=/home/ben/.openclaw/.claude claude auth login`." → A 는 보류하고 Dr. Ben 이 재로그인·확인 후 재시도. **B 타이머는 분리 처리**: parser-drain·사이드카는 claude auth 무관이라 그대로 enable; brain-drain 은 *호스트* `~/.claude` 자격(게이트웨이와 별개)에 의존 → 게이트웨이 죽음이 곧 brain-drain 죽음은 아니나, 무인 `claude -p` 비용잡이므로 host 자격도 의심되면 `~/.claude/.credentials.json` 의 refreshToken 동일 확인 권장. 자격 정상이면 조용히 통과. (게이트웨이 미실행이면 프로브 불가 — 건너뛰고 "A 토글 불가"만 보고.)
1. **A**: `cron list --all --json` 의 `enabled=false` 필터 → 각 ID `docker exec "$GW" node /app/dist/index.js cron enable <id> --token "$TOK"` 순차. ⚠️ 잡이 **하나도 없으면**(스토어 빈 상태) `cron add` 로 재등록(정의는 §참고). 단 add 전 `cron list --all` 로 동명 잡 중복 없는지 확인 — 있으면 add 말고 그 잡 enable (중복 누적 방지).
2. **B**: 호스트 타이머 전부 → `systemctl --user enable --now <timer>` (`--now` 가 OnActiveSec 시드 발화 → 체인 시작).
3. **C — Sheet 마커 갱신 (§C)**: 이 PC 의 host 행을 `<host>|on|<now>` 로 *자기 행만* 갱신(없으면 append). gog 실패 시 경고만(non-fatal).
4. 결과 표 + 요약 ("이 PC: cron N개 enable + 타이머 M개 활성 · 마커 <host>→on").

## 안전 규칙

- 이 명령은 **현재 PC** 만 토글 — cron `enabled`·타이머 enable 둘 다 **머신별 로컬**(동기 대상 아님). 다른 PC 상태는 **§C Google Sheet 마커(gog, 즉시 클라우드)로 *읽어* 확인**(토글은 여전히 현재 PC 만, 마커는 가시성 전용 — 동기되는 건 *상태 기록*이지 *토글*이 아니다). 마커 쓰기는 **자기 host 행만** — 남의 행 절대 안 건드림. gog 실패는 non-fatal(토글 우선).
- **조회 = SQLite 권위**(`cron list --all --json` 또는 `cron_jobs` 테이블 read·systemctl is-active), **변경만 정식경로**(docker exec cron / systemctl). SQLite 를 수동편집하지 말 것 — `cron <verb>` CLI 가 데몬 정합 보장. ⚠️ **구 `~/.openclaw/cron/jobs.json` 은 폐기됨**(2026.6.8 SQLite cutover, `.migrated` 잔존) — 이걸 읽으면 매번 "유실" 오판 → 중복 재등록 루프(2026-06-24 규명).
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

## 알림을 *지금* 받고 싶을 때 (토글이 아니라 발사)

`/cron` 은 **자동발화 on/off** 다. "지금 상태를 한 통 보내줘"는 다른 일이며, 알림마다 1회성 호출이 따로 있다 —
아침 헬스체크 `HEALTH_FORCE=1 brain-health.sh`, 밀린 파싱 실패 `parser-drain.sh --alert-only` 등.

**권위·명령 전문** = vault [`02_areas/brain-system/README.md` §알림 수동 발사](../../projects/2nd-brain-vault/knowledge/02_areas/brain-system/README.md).
(cron 을 꺼 둔 PC 에서도 1회성 발사는 된다 — 끄는 건 *반복 발화*지 명령 자체가 아니다.)

## 참고

- **OpenClaw cron 스토어 원본** = `~/.openclaw/state/openclaw.sqlite` 의 `cron_jobs` 테이블 (PC별·미동기, bind-mount 영속). **2026.6.8 에 평면 `jobs.json` → SQLite 로 cutover** — 구 `~/.openclaw/cron/jobs.json` 은 `jobs.json.migrated`(+`.bak`)로 남았으나 **죽은 잔존물**(읽지 말 것). (이전 변천: `openclaw-config/cron/jobs.json.template` → 2026-05-25 폐기 → `jobs.json` 원본 → 2026.6.8 SQLite.)
- **gmail-label-actions-poll 재등록 정의** (스토어가 비었을 때 §on 1 에서 사용): `cron add --agent main --name gmail-label-actions-poll --cron "*/30 * * * *" --session isolated --wake now --message "/gmail-label-actions" --channel telegram --to 8669227844 --no-deliver --token "$TOK"`. (`--no-deliver`=조용 모드 = gmail-report off 상태. 보고 켜려면 `gmail-report` 스킬 또는 `cron edit --announce`.)
- **parser-drain 타이머 정의** = `~/.config/systemd/user/parser-drain.{service,timer}` (PC별). 본체·설치는 `2nd-brain/docker/parser-drain/` (`cd ~/projects/2nd-brain/docker && make install-parser-drain`). extract(듀얼 파싱).
- **brain-drain 타이머 정의** = `~/.config/systemd/user/brain-drain.{service,timer}` (PC별). 본체·설치는 `2nd-brain/automation/brain-drain/` (`make install-brain-drain`). refine+brainify 무인 드레인.
- **brain-health 타이머 정의** = `~/.config/systemd/user/brain-health.{service,timer}` (PC별 심링크). 본체·설치는 `2nd-brain/automation/health/`(README 참조). 매일 06:40 KST 29항목 점검 → 텔레그램 아침 보고. 머신별 설정은 `~/.config/2nd-brain/health.env`(git 미추적).
- **사이드카 타이머 정의** = `~/.config/systemd/user/openclaw-*sidecar*.{service,timer}` (PC별). 사이드카 compose·이미지는 `2nd-brain/docker/webmail-sidecar/`.
- 실제 활성 상태(cron `enabled` / 타이머 enable)는 **PC 마다 독립** — git/SyncThing 동기 대상 아님.
- **cron 상태 마커** = **Google Sheet** spreadsheetId `1eXlbYvKVtAo5GEKTBjFp9Uu3t1XDFxee4_vuQFxRfBw`(탭 `cron-status`, gog, 계정 kimbi.kirams). 호스트당 1행(`host|state|updated`), **각 PC 가 자기 행만 기록·남은 읽기**. enabled 토글은 미동기지만 *각 PC 발화 여부* 가시성은 이 advisory 마커로 공유 — /cron on/off 가 자기 행 갱신(§C), status 가 전부 읽음. (변천: 2026-05-27 신설 단일 `active_host`→PC별 항목(vault SyncThing `cron-active-host.md`)→**gog Google Sheet 로 전환**(SyncThing 전파지연·잠들기 race 회피, 동기 클라우드 write). 구 vault 파일은 Sheet 포인터로 격하.)
- 타이머는 모두 `OnActiveSec`(시드) + `OnUnitInactiveSec`(no-overlap 재무장) 패턴 — `enable --now`/재로드마다 재무장(인라인 주석 금지: bad-setting 으로 조용히 폐기됨, 2026-05-26 결함 교훈).
- 향후 사이드카 추가(society-watch 등) 시 `openclaw-*sidecar*.timer` 패턴이 자동 포함. parser-drain·brain-drain 외 새 호스트 드레인 추가 시 탐색 목록에 명시 추가.
