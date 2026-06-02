---
description: gmail-label-actions cron 의 텔레그램 보고를 on/off 토글. announce(보고) ↔ none(조용) 한 필드만 cron edit 으로 패치 — 스킬 코드·게이트웨이 재빌드 불필요. 처리 동작은 그대로, 알림만 좌우.
---

# /gmail-report

`gmail-label-actions-poll` cron 이 **매 실행 후 텔레그램으로 결과를 보고할지** 를 토글한다. 보고 여부는 cron job 의 `delivery.mode` 한 필드가 좌우:

- `announce` → 에이전트 턴 결과(요약)를 텔레그램 타깃에 전송 = **보고**
- `none` → runner fallback 전송 없음 = **조용** (처리·캡처·후속작업은 그대로 수행, 알림만 안 옴)

> 캡처/후속작업 자체는 `delivery.mode` 와 무관 — 메일 처리는 항상 돈다. 이 토글은 *알림*만 끈다. 결과 확인은 OpenClaw 세션 로그(`cron runs`) 또는 Gmail 라벨 변화(→ `9 완료`)로 가능.

## 인자

- `(없음)` 또는 `status` — 현재 delivery.mode 표시
- `on` — 보고 켜기 (`--announce`)
- `off` — 조용히 (`--no-deliver`)

## 절차

### 0. 게이트웨이 + 토큰 + job id

```bash
GW=$(docker ps --filter name=openclaw-gateway --format '{{.Names}}' | head -1)
# 게이트웨이 미실행이면($GW 빈값) cron 토글 불가 → 그 사실 보고하고 종료.
TOK=$(docker exec "$GW" python3 -c "import json;print(json.load(open('/home/node/.openclaw/openclaw.json'))['gateway']['auth']['token'])")
# job id 는 하드코딩 말고 이름으로 조회(호스트 jobs.json 직접 읽기 — 빠름).
ID=$(python3 -c "import json;d=json.load(open('$HOME/.openclaw/cron/jobs.json'));print(next(j['id'] for j in (d if isinstance(d,list) else d.get('jobs',[])) if j.get('name')=='gmail-label-actions-poll'))")
```

### status

`cron get <id>` 으로 현재 delivery 확인 후 `보고(announce)` / `조용(none)` 으로 해석해 1줄 보고:

```bash
docker exec "$GW" node /app/dist/index.js cron get "$ID" --token "$TOK" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); m=d.get('delivery',{}).get('mode'); print('현재:', '보고(announce)' if m=='announce' else '조용(none)' if m=='none' else m)"
```

### on (보고 켜기)

```bash
docker exec "$GW" node /app/dist/index.js cron edit "$ID" --announce --token "$TOK"
```

> `--announce` 는 mode 만 바꾸고 기존 `channel`(telegram)·`to`(chatId) 는 보존 → 텔레그램 전송 복원. (만약 채널/타깃이 비어 있으면 `--channel telegram --to <chatId>` 추가.)

### off (조용히)

```bash
docker exec "$GW" node /app/dist/index.js cron edit "$ID" --no-deliver --token "$TOK"
```

### 보고

변경 후 패치 결과의 `delivery.mode` 를 읽어 `보고→조용` / `조용→보고` 로 1줄 확인.

## 안전 규칙

- **변경은 `cron edit` CLI 정식 경로** — jobs.json 의 `delivery` 수동편집 금지(데몬 정합은 CLI 가 보장).
- 게이트웨이 미실행이면 토글 불가 → 그 사실만 보고.
- 이 토글은 **현재 PC 게이트웨이** 의 job 만 바꾼다. cron `enabled`(발화 여부)는 `/cron` 소관 — 별개. 보고 토글은 발화에 영향 없음.
- 처리 동작·비가역 후속작업(라벨·일정·할일·초안)은 `delivery.mode` 와 무관 — off 로 둬도 메일은 정상 처리된다.

## 참고

- delivery 모드 3종: `announce`(보고) / `webhook`(URL POST) / `none`(조용). 이 명령은 announce↔none 만 다룬다.
- OpenClaw `cron edit` 전체 플래그: `docker exec <gw> node /app/dist/index.js cron edit --help`.
- 관련: 발화 on/off 는 `/cron`, 스킬 정의는 `~/.openclaw/workspace/skills/gmail-label-actions/`.
