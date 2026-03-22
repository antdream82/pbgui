# PB7 Hyperliquid HIP-3 stale ghost / lifecycle / margin-mode 인계 요약

## 정정 메모 (2026-03-22)

이 문서의 `335406020602` 관련 해석은 현재 기준으로는 더 이상 source-of-truth가 아니다.

이후 추가 조사로 확인된 사실:

- raw Hyperliquid `spotMeta` 기준 `@107 = HYPE/USDC`
- raw `openOrders` 기준 `oid=335406020602` 는 실제 exchange live order
- PB7 current live mode(`swap + hip3`)에서는 CCXT market namespace collision 때문에
  raw `@107`가 futures namespace의 `ALT/USDC:USDC`로 잘못 정규화되었다

즉 `335406020602`는 진짜 stale futures ghost order라기보다
**spot order가 futures bot state에 잘못 유입된 케이스**였다.

따라서 현재 우선 문서는:

- [PB7_Hyperliquid_CCXT_spot_namespace_mitigation_handoff.md](/app/pbgui/data/PB7_Hyperliquid_CCXT_spot_namespace_mitigation_handoff.md)

이 문서의 tombstone 관련 서술은 **당시 관찰 기준 기록**으로만 읽어야 하며,
현재 결론으로는 `335406020602`에 대한 대표 해석으로 사용하면 안 된다.

## 현재 upstream과 다른 최종 유지 패치

- `fetch_open_orders()` 에서 raw `@...` spot open order는 futures state 주입 전에 skip
- cancel-gone 처리 시 local `open_orders` 제거는 `str(id)` 기준으로 정규화
- HIP-3 margin-mode 판정은 prefix 강제가 아니라 market metadata 기반 유지
- `match_check` 진단 로그는 최종 pair/unmatched 결과만 남기고 중간 후보 비교 noise는 줄임

## 대상 환경

- exchange: Hyperliquid
- user: `hyper`
- 주요 로그 파일:
  `/app/pbgui/data/run_v7/hyper/passivbot.log`
- 주요 코드 파일:
  `/app/pb7/src/exchanges/hyperliquid.py`
- 관련 테스트 파일:
  `/app/pb7/tests/test_stock_perps.py`
- tombstone 파일:
  `/app/pb7/caches/hyperliquid/hyper_cancel_gone_tombstones.json`

## 트랙 A: `335406020602` 초기 가설과 이후 정정

주의:
- 아래 섹션은 **당시 조사 시점의 가설/조치 기록**이다.
- 현재 재평가 기준으로 `335406020602`는 stale ALT ghost가 아니라
  `@107` spot order 오염 사례로 분류된다.

### 문제 대상

- symbol: `ALT/USDC:USDC`
- stale ghost order id: `335406020602`
- 주문 내용: buy long `0.56 @ 29.0`

### 원래 증상

Hyperliquid에서 exchange 측에서는 이미 canceled/filled(or never placed) 취급인 주문이 `fetch_open_orders() -> normalize -> update_open_orders()` rebuild 경로를 통해 다시 local `open_orders`에 유입되었다.

오염 경로:

- `open_orders_after_update`
- `open_orders_before_planning`
- `orders_old_sorted`
- `cancel_try`
- `execute_cancellation()` 에서는 already gone 처리
- 이후 다시 fetch/rebuild에서 동일 id 재유입

### 핵심 해석

문제는 2층으로 분리된다.

1. upstream ghost source
- stale entity 자체는 upstream fetch/rebuild 쪽에서 공급되었다.
- 증거:
  - 재시작 직후 `open_orders_after_update` 에서 다시 등장
  - `raw_symbol=@107`
  - `route=default:*`

2. local lifecycle amplification
- 과거에는 같은 id에 대해
  - `cancel_try`
  - `cancel_gone removed_locally=False`
  가 반복되며 local infinite cancel loop에 가까운 상태가 만들어졌다.
- 즉 loop의 직접 원인은 PB7 local open-order lifecycle 처리 실패 쪽 영향이 더 컸다.

정리:

- upstream이 stale ghost를 공급했고
- PB7가 그 ghost를 로컬에서 제대로 제거하지 못해 loop가 증폭되었다.

### 적용된 방어

1. key normalization
- marker key는 `(symbol, str(id))` 기준으로 통일
- local remove / tracked_open / planning lookup도 `str(id)` 기준으로 맞춤

2. fetch-level suppress
- `fetch_open_orders()` 에서 recent cancel-gone tombstone 대상 id suppress

3. update_open_orders() suppress
- rebuild 결과를 `self.open_orders`에 반영하기 전에 tombstone 기반으로 한 번 더 필터

4. planning-stage guard
- `_snapshot_actual_orders()` 에서도 tombstone 대상은 planning 입력으로 내려가지 않게 보조 방어

### retention

현재 운영 retention:

- `CANCEL_GONE_MARKER_MAX_AGE_MS = 21_600_000`
- 즉 6시간

과거 실험 요약:

- 15초: 실패
- 60초: 실패
- 5분: 실패
- 15분: 임계값 근처에서 재실패
- 30분: 운영 방어로 유효
- 현재는 보수적으로 6시간

### restart weakness 와 해결

과거 핵심 약점:

- marker가 메모리 기반이어서 restart 시 tombstone memory가 사라짐
- fresh startup 직후 stale id가 다시 `open_orders_after_update` 로 들어옴

이를 보완하기 위해 `_diag_cancel_gone_orders` 를 restart-safe tombstone store처럼 persist/restore 하도록 추가했다.

### tombstone persistence 구현

구현 파일:

- `/app/pb7/src/exchanges/hyperliquid.py`

구현 내용:

- 저장 대상:
  `(symbol, id) -> ts`
- 파일 경로:
  `caches/<exchange>/<user>_cancel_gone_tombstones.json`
- 실제 runtime:
  `/app/pb7/caches/hyperliquid/hyper_cancel_gone_tombstones.json`
- JSON 예:

```json
{
  "version": 1,
  "entries": [
    {"symbol": "ALT/USDC:USDC", "id": "335406020602", "ts": 1773793544689}
  ]
}
```

- load 시점:
  `HyperliquidBot.__init__()` 에서 `super().__init__(config)` 직후
- save 시점:
  `_mark_cancel_gone()` 직후
- prune:
  load 후 prune / save 전 prune
  둘 다 `CANCEL_GONE_MARKER_MAX_AGE_MS` 기준
- 저장 방식:
  temp file + `replace()` 로 atomic write
- load/save 실패 시:
  warning만 찍고 계속 진행

### restore 진단 로그

현재 코드에는 다음 로그가 있다.

- `[diag][cancel_gone_restore_enter]`
- `[diag][cancel_gone_restore]`
- `[diag][cancel_gone_restore_pruned]`
- `[diag][cancel_gone_restore_probe]`

### 당시 ALT ghost 최종 상태

실제 fresh restart 기준으로 아래가 확인되었다.

- `2026-03-18T03:58:48` restart 직후
  - `cancel_gone_restore_enter`
  - `cancel_gone_restore`
  - `cancel_gone_restore_pruned`
  - `cancel_gone_restore_probe present=True`
  가 모두 찍힘
- 이후 stale id `335406020602` 는 계속
  - `cancel_gone_refetched`
  - `cancel_gone_suppressed`
  로만 보임
- 더 이상
  - `open_orders_after_update`
  - `open_orders_before_planning`
  - `ALT/USDC:USDC` 의 `orders_old_sorted`
  - `cancel_try`
  로 재오염되지 않음

즉:

- upstream stale source는 여전히 존재
- local lifecycle 방어는 runtime + restart 모두에서 동작
- 운영 관점에서는 증상은 사실상 해결

## 트랙 B: Hyperliquid HIP-3 margin-mode 판정 문제

### 문제 배경

기존 PB7는 `xyz:` / `XYZ-` / `XYZ:` prefix만 보면 HIP-3 symbol을 사실상 모두 `isolated` 로 분류했다.

핵심 함수:

- `/app/pb7/src/exchanges/hyperliquid.py`
  - `HyperliquidBot._requires_isolated_margin()`
- `/app/pb7/src/exchanges/ccxt_bot.py`
  - `_get_margin_mode_for_symbol()`
  - `_calc_leverage_for_symbol()`

기존 동작 때문에:

- `XYZ-XYZ100/USDC:USDC` 도 항상 `isolated` 로 설정되었다.
- 로그에서도 과거에는 계속
  - `XYZ-XYZ100/USDC:USDC: margin=ok (isolated)`
  만 찍혔다.

### 왜 문제라고 판단했는가

trade[XYZ] 공식 문서상 instrument별 margin mode가 섞여 있었고,
실제 runtime market metadata도 종목별로 달랐다.

직접 확인한 runtime metadata 예:

- `XYZ-XYZ100/USDC:USDC`
  - `onlyIsolated = null`
  - `marginMode = null`
  - `maxLeverage = 30`
- `XYZ-TSLA/USDC:USDC`
  - `onlyIsolated = null`
  - `marginMode = null`
  - `maxLeverage = 10`
- `XYZ-INTC/USDC:USDC`
  - `onlyIsolated = true`
  - `marginMode = "noCross"`
  - `maxLeverage = 10`
- `XYZ-GOLD/USDC:USDC`
  - `onlyIsolated = null`
  - `marginMode = null`
  - `maxLeverage = 25`

즉:

- `INTC` 같은 종목은 명시적 isolated-only
- `XYZ100`, `TSLA`, `GOLD` 는 적어도 metadata상 prefix만으로 isolated라고 단정할 근거가 없었다

### live API 실험 결과

실계정 `hyper` 로 직접 확인했다.

1. `XYZ-XYZ100/USDC:USDC` 에 open position/open orders 가 있는 상태
- `set_margin_mode("cross", ...)` 호출 결과:
  - `Cannot switch leverage type with open position.`

2. `strict flat` 상태
- position 0
- open orders 0
- 같은 호출 결과:
  - 성공
  - 응답:

```json
{
  "status": "ok",
  "response": {
    "type": "default"
  }
}
```

즉 확정된 사실:

- `XYZ100` 은 적어도 flat 상태에서는 실제로 `cross` 전환 가능
- 기존 PB7의 “HIP-3 prefix면 무조건 isolated” 가정은 `XYZ100` 에는 맞지 않았다

### 적용한 패치

수정 파일:

- `/app/pb7/src/exchanges/hyperliquid.py`
- `/app/pb7/tests/test_stock_perps.py`
- `/app/pb7/CHANGELOG.md`

핵심 수정:

- `_requires_isolated_margin()` 를 prefix 기반 강제에서 metadata 우선 판정으로 변경

현재 isolated 판정 기준:

- `info["onlyIsolated"] == True`
- `info["isolatedOnly"] == True`
- `info["marginMode"] == "noCross"`

그 외에는 base class generic isolated flag 확인만 사용

즉 결과적으로:

- `XYZ100` -> `cross` 후보
- `INTC` -> `isolated`

### 테스트

수정 후 타깃 테스트 실행:

```bash
PYTHONPATH=src /venv_pb7/bin/python -m pytest tests/test_stock_perps.py -q -s
```

결과:

- `.................................`
- pass

추가로 문법 확인:

```bash
/venv_pb7/bin/python -m py_compile src/exchanges/hyperliquid.py
/venv_pb7/bin/python -m py_compile tests/test_stock_perps.py
```

둘 다 통과

### 실제 운영 로그 검증

실제 restart:

- `2026-03-18T14:11:30`

그 직후 로그:

- `2026-03-18T14:11:56 INFO [hyperliquid] XYZ-XYZ100/USDC:USDC: margin=ok (cross)`

즉 이번 패치가 실제 runtime에서 적용된 것이 확인되었다.

과거에는 계속

- `margin=ok (isolated)`

였고, 이번 재시작 후에는

- `margin=ok (cross)`

로 바뀌었다.

### cross 전환 후 runtime 상태

현재까지 로그상:

- `XYZ100` 이 `cross` 로 설정된 뒤 다시 `isolated` 로 되돌아간 로그 없음
- `error setting ... mode` 없음
- `order plan summary` 는 정상 진행
- `orders_new_sorted` / `orders_old_sorted` 흐름도 계속 이어짐

### balance / equity 영향 점검

관련 코드:

- `/app/pb7/src/exchanges/hyperliquid.py`
  - `_fetch_positions_and_balance()`

현재 equity 계산 구조:

- default `fetch_balance()` 의 `accountValue`
- plus relevant HIP-3 dex들의 `accountValue`

로그상 현재까지는:

- `XYZ100 cross` 전환 후 balance/equity가 비정상적으로 튀는 증거 없음
- startup 직후:
  - `balance raw 16418.419819`
  - `equity 16418.4198`
- cross 전환 후:
  - `balance raw 16418.303335`
  - `equity 16418.3033`

즉 현재까지는 overcount / undercount 징후 없음

다만 잠재 리스크:

- future runtime에서 default accountValue와 dex accountValue가 동시에 비제로로 잡히는 방식이면
  current 합산 로직이 double-count risk를 가질 수 있음
- 현재 로그에서는 그런 증거는 아직 없음

운영 관찰 포인트:

```bash
grep -E "\\[balance\\]|\\[health\\]|XYZ-XYZ100/USDC:USDC: margin=ok \\(cross\\)" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 100
```

의심 신호:

- `margin=ok (cross)` 이후 equity가 balance보다 비정상적으로 크게 점프
- 포지션/주문 변화가 작은데 equity만 크게 튐
- balance/equity 관련 error 로그

## 현재 시점 종합 결론

### `335406020602` / ALT ghost / tombstone

- 현재 결론으로는 `335406020602`는 stale ALT ghost가 아니라
  **raw spot `@107` order(HYPE/USDC)가 CCXT namespace collision로 잘못 정규화된 케이스**
- 따라서 이 사례를 tombstone 성공 사례로 계속 사용하면 안 됨
- 현재 live safety 완화는 tombstone이 아니라
  **raw `@...` spot open order skip** 이 source-side 차단 역할을 함

### XYZ100 margin-mode

- 기존 PB7의 “HIP-3 prefix면 isolated” 가정은 과도했고 `XYZ100` 에는 부정확했다
- 실제 exchange metadata와 live API 실험 결과를 반영해
  metadata 기반 판정으로 수정
- 실제 restart 이후 `margin=ok (cross)` 확인

### 현재 운영적으로 볼 때

- ALT stale ghost order 문제: 방어 완료 상태로 보임
- `XYZ100 cross` 전환 패치: 실제 runtime에서 적용 확인
- balance/equity: 현재까지 이상 징후 없음, 다만 continued observation 권장

## 다음에 확인할 것

1. `XYZ100` 이 계속 `margin=ok (cross)` 상태를 유지하는지
2. `XYZ100` 체결 이후에도 balance/equity가 정상 범위인지
3. raw spot order `335406020602` 가 계속 exchange fetch에는 보이더라도
   - `[diag][skip_spot_open_order]`
   로만 끝나는지
4. 다음 PB7 업그레이드 시 아래가 유지되는지
   - raw `@...` spot open-order skip
   - metadata-based margin-mode detection

## 유용한 확인 명령

### `335406020602` / raw spot order

```bash
grep "335406020602" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 200
```

```bash
grep -E "skip_spot_open_order|335406020602|cancel_gone_refetched|cancel_gone_suppressed|cancel_gone_restore" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 250
```

### XYZ100 margin mode

```bash
grep -E "XYZ-XYZ100/USDC:USDC: margin=|XYZ-XYZ100/USDC:USDC.*cross|XYZ-XYZ100/USDC:USDC.*isolated" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 80
```

### balance / equity

```bash
grep -E "\\[balance\\]|\\[health\\]|XYZ-XYZ100/USDC:USDC: margin=ok \\(cross\\)" /app/pbgui/data/run_v7/hyper/passivbot.log | tail -n 120
```

## 이번 세션 최종 요약

- `335406020602`는 현재 결론으로 stale ALT futures ghost가 아니라 raw spot `@107` order(HYPE/USDC) 오염 사례다.
- 이 문제의 최소 안전 완화는 tombstone persist/suppress가 아니라 raw `@...` spot open order를 futures bot ingestion에서 제외하는 것이다.
- Hyperliquid HIP-3 `XYZ100` 은 flat 상태에서 실제 `cross` 전환이 가능하며, PB7는 이를 반영하도록 metadata 기반 margin-mode 판정으로 수정되었다. 실제 restart 이후 운영 로그에서 `XYZ-XYZ100/USDC:USDC: margin=ok (cross)` 가 확인되었다.

## Current local divergence from upstream

Currently kept local changes:

1. In Hyperliquid open-order ingestion, raw spot-style `@...` orders are skipped before futures normalization/state injection.
2. Local open-order removal on cancel-gone normalizes ids via `str(id)`.
3. HIP-3 margin-mode detection uses market metadata (`onlyIsolated` / `isolatedOnly` / `marginMode=="noCross"`) instead of prefix-only forcing.

All temporary tombstone persistence/suppress changes added during the `335406020602` investigation were later reverted.
