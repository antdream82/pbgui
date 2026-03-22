# PB7 Hyperliquid CCXT spot namespace collision 완화 인계

## 요약

이번 패치는 PB7 내부 tombstone/lifecycle 버그를 고친 것이 아니라,  
**Hyperliquid raw spot order symbol(`@...`)이 CCXT market namespace와 충돌하면서 futures bot state를 오염시키던 문제를 막기 위한 최소 완화**다.

핵심 결론:

- raw Hyperliquid truth 기준 `@107 = HYPE/USDC`
- 그러나 PB7 현재 live mode(`swap + hip3`)에서 CCXT `load_markets()`는 `107 = ALT/USDC:USDC`를 제공
- raw `openOrders`가 돌려준 spot-style `@107`가 futures namespace로 잘못 정규화되며
  - `open_orders`
  - `cancel_gone`
  - `tombstone`
  - 진단 로그
  를 오염시켰다
- 따라서 immediate live safety 관점의 가장 작은 완화는  
  **raw symbol이 `@...`인 open order를 futures bot open-order ingestion에서 제외**하는 것이다

## 왜 upstream과 다르게 패치했는가

upstream는 Hyperliquid spot order `@index`가 현재 bot mode에서 futures state로 섞여 들어오는 상황을 따로 막지 않는다.

이번 환경에서는 다음 조건이 동시에 성립했다.

1. bot은 `swap + hip3` markets만 로드
2. raw exchange `openOrders`는 spot-style `coin="@107"`를 반환
3. raw exchange `spotMeta` truth는 `@107 = HYPE/USDC`
4. 같은 런타임에서 CCXT market metadata는 `107 = ALT/USDC:USDC`

즉 이건 단순 사용 오류가 아니라 **CCXT namespace/mode mismatch**였다.

upstream를 그대로 두면:

- raw spot order가 futures symbol로 잘못 정규화될 수 있고
- 그 주문이 local open-order lifecycle 전체를 오염시킬 수 있었다

live safety를 위해 upstream fix를 기다리지 않고 local mitigation을 넣었다.

## 어떤 방법으로 패치했는가

### 목표

동작을 넓게 바꾸지 않고, 오염 유입만 차단한다.

### 실제 변경

파일:

- `/app/pb7/src/exchanges/hyperliquid.py`

함수:

- `HyperliquidBot.fetch_open_orders()`

로직:

1. raw exchange order를 fetch
2. `raw_symbol = order["symbol"]`
3. `raw_symbol`이 문자열이고 `@`로 시작하면:
   - concise diagnostic log 남김
   - 해당 order는 `continue`
4. 그 외 주문만 기존 futures normalization 경로 진행

의도:

- current PB7 live bot은 futures bot이므로
- raw spot order를 futures open-order state에 넣지 않는다
- 일반 futures/hip3 order 동작은 그대로 유지한다

## 정확한 패치 범위

### 유지한 것

- Hyperliquid margin-mode metadata 패치
- Hyperliquid balance denominator 수정
- order churn 완화용 tolerance 수정
- cancel-gone 시 local `open_orders` 제거 정규화(`str(id)` 기준)

### 되돌린 것

처음에는 `335406020602`를 stale ghost futures order로 오인해서 다음 방어가 들어갔다.

- cancel-gone tombstone persistence
- restart restore
- fetch/update/planning 단계 suppress
- 관련 진단 로그

하지만 나중에 확인된 사실은:

- `335406020602`는 실제 exchange open order
- raw symbol은 `@107`
- raw `spotMeta` 기준 `@107 = HYPE/USDC`

즉 원래 문제는 stale ghost lifecycle이 아니라  
**spot order가 futures namespace로 잘못 들어온 것**이었다.

그래서 tombstone 계열은 제거했고, 오염 source를 ingress에서 차단하는 방향으로 정리했다.

## 근거 데이터

### raw Hyperliquid truth

- `POST /info {"type":"spotMeta"}`
  - universe index `107`
  - tokens `[150, 0]`
  - token `150 = HYPE`
  - token `0 = USDC`
  - 따라서 `@107 = HYPE/USDC`

### raw open order

- `POST /info {"type":"openOrders","user":"..."}`
  - `oid = 335406020602`
  - `coin = @107`
  - `limitPx = 29.0`
  - `sz = 0.56`

### CCXT mode 분류 테스트

관찰:

- `spot_only`
  - `HYPE/USDC`
  - `id="@107"`
- `swap_only`
  - `ALT/USDC:USDC`
  - `id="107"`
  - `baseId=107`
- `swap + hip3`
  - 동일하게 `107 = ALT`
- `spot + swap`
  - 둘 다 공존

결론:

- CCXT 자체가 `spot @107`과 `swap 107`을 서로 다른 namespace로 다룸
- current PB7 live mode는 spot market을 로드하지 않으므로
- raw `@107` order가 들어오면 safe normalization이 불가능

## 왜 이 완화가 안전한가

1. 비-`@...` 주문은 기존 경로를 그대로 탄다
2. Rust strategy math를 건드리지 않는다
3. order generation / risk / unstuck 로직을 건드리지 않는다
4. 잘못된 symbol remap보다 **명시적 skip이 더 fail-safe**다

즉 이 변경은 strategy behavior 변경이 아니라  
**잘못된 external state 유입 차단**이다.

## 테스트

관련 테스트 파일:

- `/app/pb7/tests/test_stock_perps.py`

추가/조정된 테스트:

1. raw `@107` order는 futures open-order state에 들어가지 않음
2. raw `@107` spot order가 있어도 normal futures order는 그대로 유지됨

추가로 tombstone 계열 테스트는 이번 재분석에 맞춰 제거/축소했다.

## 런타임 검증 포인트

패치 반영 후 기대 로그:

- 반복적으로 exchange fetch에는 보일 수 있음
  - `raw_symbol=@107`
- 하지만 이후엔
  - `[diag][skip_spot_open_order] raw_symbol=@107 id=335406020602 ...`
  로 끝나야 함
- 더 이상
  - `cancel_gone_refetched`
  - `cancel_gone_suppressed`
  - `cancel_gone_restore`
  경로로 내려가면 안 됨

## 로컬 확인 명령

### raw exchange open order

```bash
python - <<'PY'
import requests
user='YOUR_USER_ADDRESS'
print(requests.post(
    'https://api.hyperliquid.xyz/info',
    json={'type': 'openOrders', 'user': user},
    timeout=20,
).text)
PY
```

### recent PB7 log check

```bash
rg -n "skip_spot_open_order|335406020602|cancel_gone_refetched|cancel_gone_suppressed|cancel_gone_restore" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 200
```

## 장기적으로 더 좋은 구조

이번 패치는 intentional minimal mitigation이다.

장기적으로는:

- Hyperliquid raw `@index` spot symbol resolution에 대해
- CCXT `baseId`/market cache를 그대로 신뢰하지 않고
- raw `spotMeta`를 source-of-truth로 쓰는 별도 resolver를 두는 것이 맞다

하지만 current live safety 관점에서는  
**spot `@...` open order skip**이 가장 작은 안전 패치다.
