# システム全体仕様書（AI漁師 / WAX・決済ゲート・ブラケット注文の実体）

生成: 2026-06-14 / 対象: `backend/src/`（凍結5＝activeExit/fishermanTrader/fishermanForecast/waxTrader/waxForecast）。
本書は客観的事実とソースコードの生コードのみで構成する（要約・評価・言い訳を含まない）。引用は file:line を付す。

---

## 1. エージェント別 稼働定義（AI漁師 vs WAX）

| 項目 | **AI漁師（fisherman）** | **WAX** |
|---|---|---|
| コンセプト（コード根拠） | 「AI漁師」=指標トレード専門・半自動。`fishermanForecast.ts:84`「あなたは USD/JPY を張る『AI漁師』。これは**ギャンブル的な決め打ち**」。`fishermanForecast.ts:28` `direction: 'UP'|'DOWN'`（WAIT廃止＝必ず方向を決め打つ） | 「会長 sada の分身」。`waxForecast.ts` / `waxTrader.ts:3`「AI漁師(fisherman)と WAX(会長の分身)が【同じ器】を使う」。S/R(市場の真理)主体の波乗り＋大口の動き |
| 狙う波 | 指標発表〜直後の初動（イベント駆動）。`fishermanTrader.ts:TRIGGER_LEAD_MS=10分前`、`POST_EVENT_GRACE_MS=30分`（発表時刻±窓） | 「大口の大波の時間帯」。`waxForecast.ts:39` `bigPlayerWindow: boolean // 今が大口の大波の時間帯か`、`waxForecast.ts:56 bigPlayerWindow()` |
| エントリーのトリガー | ①指標イベント窓（`fisherman_forecasts`＝指標APIバッチ＋web検索でforecast生成）②初動ベクトル（`entryGuardian`/`entryScout` のvector検知）③レジサポ幾何（`entryGuardian` のpark/band＝S/Rターンで現値±1.5p狭帯を書き凍結tickが約定）④報酬ゲート（`entryGuardian.rewardGate`＝報酬≥瞬間spread+3p） | 委任（「今日はWAXに任せる」）中のみ、大口時間帯にWAX自身がエントリー。`waxTrader.ts` 委任ロジック＋`judgeWaxEntry`（価格優先のAI判断） |
| AI（Sonnet 4.6）の役割 | **入口バイアスのみ**。`fishermanEntryJudge.ts:15 JUDGE_MODEL='claude-sonnet-4-6'`。トリガー窓入りにarmごと1回 `bias: 'buy'|'sell'|'both'|'skip'` を返す（発射はアルゴ＝AIをtick毎に呼ばない）。建玉中は `positionCoach.ts`（D-1 船長Sonnet・出口を早める方向のみ）だが**機械防衛が下層・不変** | **入口判断＋出口判断の両方をAIが担う**。`waxForecast.ts:24 WAX_MODEL='claude-sonnet-4-6'`、`waxForecast.ts:313 judgeWaxEntry()`（建値圏到達時に張る/見送り/反転をAI判断）、`judgeWaxExit`（確保/伸ばす/切る） |
| 約定後の能動決済instance設定 | `fishermanTrader.ts:41` `fishInstance = createActiveInstance({ certainExit:{ targetTouchTP:false … }, stopLoss: 予想SL到達でcut })`＝**目標タッチ利確を取らず天井トレイル維持** | `waxTrader.ts:69` `waxInstance = createActiveInstance({ certainExit:{ targetTouchTP:true … }, stopLoss: M5実体S/R±8p突破&ヒゲ実体比2倍未満でcut })`＝**目標タッチで取る** |

### 1.1 instance 定義（生コード）

`ai/fishermanTrader.ts:41-49`
```ts
const fishInstance = createActiveInstance({
  tag: FISHERMAN_TAG, table: 'fisherman_forecasts',
  trailArmPips: TRAIL_ARM_PIPS, trailGivebackPips: TRAIL_GIVEBACK_PIPS,
  certainExit: { targetTouchTP: false, breakevenArmPips: FISH_BE_ARM_PIPS, breakevenFloorPips: FISH_BE_FLOOR_PIPS },
  stopLoss: ({ buy, cur, row }) => {
    const sl = row['stopLoss']
    if (typeof sl === 'number' && Number.isFinite(sl) && ((buy && cur <= sl) || (!buy && cur >= sl))) return { cut: true }
    return { cut: false }
  },
})
```

`ai/waxTrader.ts:69-80`
```ts
const waxInstance = createActiveInstance({
  tag: WAX_TAG, table: 'wax_forecasts',
  trailArmPips: TRAIL_ARM_PIPS, trailGivebackPips: TRAIL_GIVEBACK_PIPS,
  certainExit: { targetTouchTP: true, breakevenArmPips: WAX_BE_ARM_PIPS, breakevenFloorPips: WAX_BE_FLOOR_PIPS },
  stopLoss: ({ buy, candles, row }) => {
    const sr = Number(row['srLevel'])
    if (!Number.isFinite(sr) || candles.length === 0) return { cut: false }
    const c = candles.length >= 2 ? candles[candles.length - 2]! : candles[candles.length - 1]!
    const body = Math.abs(c.close - c.open)
    if (!(body > 0)) return { cut: false }
    // 買い: サポート-8pips下抜け & 下ヒゲ<実体×2(突き抜け)→損切り。下ヒゲ≥実体×2(跳ね返し) or 実体がS内側→耐える
    …(M5実体S/R±8p突破&ヒゲ判定)…
  },
})
```

### 1.2 AI漁師の入口バイアス（Sonnet 4.6・生コード抜粋）
`entry/fishermanEntryJudge.ts:15`
```ts
const JUDGE_MODEL = 'claude-sonnet-4-6'
const MAX_JUDGE_PER_DAY = 30
```
プロンプト（`entry/fishermanEntryJudge.ts:runJudge` 内）:
```
あなたはUSDJPYスキャルの入口判断。指標発表の直前、走り出した初動に順張りで乗るかの方向バイアスだけを決める。
イベント: {eventName} / AI予想方向: {買い|売り} / 現値: {price}
直近10分: ネット{net10}p・レンジ{range10}p
レジーム: {regime}
回答はJSONのみ: {"bias":"buy|sell|both|skip","reason":"30字以内"}
基準: 価格優先。予想と逆に走り出しているなら逆バイアス可。方向感なし=both。動くべきでない(危険/薄商い)時のみskip。
```

---

## 2. 共通決済ゲート（exitGuardian / activeExit）の構造

### 2.1 ルーティング（条件分岐の優先順位）

毎tick（autotrader）で、漁師・WAX建玉は**2つの能動決済層**を通過する（汎用autotrader決済は両者を除外）。

1. **汎用autotrader決済の除外** — `autotrader.ts:624`
   ```ts
   if (_ag.includes('fisherman') || _ag.includes('wax')) continue   // ★漁師・WAXポジは自身が能動決済(天井で取る)。他系から触らない
   ```
2. **第1層 activeExit（凍結・共有器）** — `fishermanTrader.fishermanTick`（`autotrader.ts:1169` で起動）の中で `ai/fishermanTrader.ts:242`：
   ```ts
   await manageActiveExits(fishInstance, ctx.positions, ctx.closePosition)
   ```
   WAXは `waxTrader` が `waxInstance` で同関数を呼ぶ（同じ `activeExit.manageActiveExits` 器）。
3. **第2層 exitGuardian（非凍結・全建玉共通）** — `autotrader.ts:1221`：
   ```ts
   if (px > 0) await (require('./entry/exitGuardian') as typeof import('./entry/exitGuardian')).exitTick({ positions: ps, price: px, m1: m1x, m5: m5x, m15: m15x, h1: h1x, h4: h4x, closePosition })
   ```
   `exitGuardian.ts:11` kill flag `exit_guardian='off'` → 「一切決済しない(凍結activeExitのみの従来動作)」。すなわち exitGuardian は activeExit の上に重なる第2の能動層。

→ **同一の closePosition（backend→MT5 /close）に依存する能動決済が、漁師に対して二層（activeExit ＋ exitGuardian）走る。** WAXも同様（activeExit ＋ exitGuardian）。判断の中身は各層が独立に持つ（下記生コード）。

### 2.2 `entry/exitGuardian.ts` `exitTick` の分岐ロジック（生コード）

対象選別とSL/TP/トレイル/波死亡の二択判定（優先順）:
```ts
export async function exitTick(ctx: { positions: MT5Position[]; price: number; m1: CandleData[]; m5: CandleData[]; m15: CandleData[]; h1: CandleData[]; h4: CandleData[]; closePosition: CloseFn }): Promise<void> {
  …
  for (const p of ctx.positions) {
    const agent = String((p as unknown as Record<string, unknown>)['agent'] ?? '')
    if (!agent.includes('fisherman') && !agent.includes('wax')) continue   // 対象=漁師/WAXのみ
    …
    // 全判断を「決済可能サイドの実価格(current_price: 買→bid/売→ask)」で行う
    const curRaw = Number((p as unknown as Record<string, unknown>)['current_price'])
    const cur = Number.isFinite(curRaw) && curRaw > 0 ? curRaw : ctx.price
    const prof = (buy ? cur - entry : entry - cur) / PIP
    if (prof > s.peak) s.peak = prof
    …
    // ── 二択判定(優先順) ──
    if (cv.closeNow) {                                   // 1. AI船長「今すぐ利確」
      close = `AI船長「今すぐ利確」→確定 …`
    } else if (shelf != null && prof > 0 && (buy ? cur >= shelf - EXIT_CFG.shelfTakePips * PIP : cur <= shelf + EXIT_CFG.shelfTakePips * PIP)) {
      close = `棚到達(${shelf.toFixed(3)}手前)→確定 …`          // 2. 棚到達(対向S/R手前1p)
    } else if (s.peak >= EXIT_CFG.trailArmPips && s.peak - prof >= giveEff) {
      close = `精密トレイル(+${s.peak}p→${giveEff}p戻り…)→確定 …`  // 3. 精密トレイル(peak≥2p・戻り1.2p/AI警戒0.6p)
    } else if (s.peak >= EXIT_CFG.beArmPips && prof <= EXIT_CFG.beFloorPips) {
      close = `建値ストップ(+${s.peak}p後に建値圏)→確定 …`        // 4. 建値ストップ(peak≥3p後 prof≤0.2p)
    } else if (net3 <= -agTh && s.peak >= 1.0) {
      close = `波死亡(直近3本が${Math.abs(net3)}p逆行)→即フラット …` // 5. 波死亡(逆行≥0.8p & peak≥1p)
    } else if (ageSec > EXIT_CFG.deadMinAgeSec && range5 <= rgTh && prof < s.peak - 0.5) {
      close = `波死亡(M1レンジ${range5}pに収縮)→即フラット …`       // 6. 波死亡(建後180s超 & 5本レンジ≤1.2p)
    } else { … 継続(実況のみ) … }
    if (close && enabled) {
      try { await ctx.closePosition(ticket); … }
      catch (e) { console.warn(`[exitG] close失敗 tk${ticket}:`, String(e)) }
    }
  }
}
```
閾値 `EXIT_CFG`（`exitGuardian.ts:19`）: `trailArmPips 2.0 / trailGivebackPips 1.2 / beArmPips 3.0 / beFloorPips 0.2 / deadAgainstPips 0.8 / deadRangePips 1.2 / deadMinAgeSec 180 / shelfTakePips 1.0`。

### 2.3 `ai/activeExit.ts` `manageActiveExits` の分岐ロジック（生コード・共有器＝漁師/WAX共通）
```ts
export async function manageActiveExits(inst, positions, closePosition, candles = []) {
  const mine = posOfInstance(inst, positions)   // comment先頭タグ(fisherman/wax)で自分の建玉だけ
  for (const p of mine) {
    const row = db.prepare(`SELECT * FROM ${inst.table} WHERE ticket=? AND filledAt IS NOT NULL AND status != 'closed' …`).get(p.ticket)
    …
    const profitPips = buy ? (cur - entry) / PIP : (entry - cur) / PIP
    peak = buy ? Math.max(peak, cur) : Math.min(peak, cur)
    const peakProfitPips = buy ? (peak - entry) / PIP : (entry - peak) / PIP
    // ★損切り(instance注入: 漁師=予想SL到達 / WAX=M5実体S/R突破&ヒゲ)
    let sl = inst.stopLoss({ buy, entry, cur, row, candles })
    if (sl.cut) { await closePosition(p.ticket); continue }                       // (A) 能動損切り
    const ce = inst.certainExit
    if (ce) {
      if (ce.targetTouchTP) {                                                     // (B) 目標タッチ利確(WAXのみ true)
        const tp = Number(row['takeProfit'])
        const targetPips = buy ? (tp - entry) / PIP : (entry - tp) / PIP
        if (targetPips > 0 && profitPips >= targetPips) { await closePosition(p.ticket); continue }
      }
      if (ce.breakevenArmPips > 0 && peakProfitPips >= ce.breakevenArmPips && profitPips <= ce.breakevenFloorPips) {
        await closePosition(p.ticket); continue                                   // (C) 建値バックストップ(peak≥8p後 建値圏)
      }
    }
    if (peakProfitPips >= inst.trailArmPips) {                                     // (D) 天井トレイル(+ARMで開始→GIVEBACK戻りで決済)
      const giveback = peakProfitPips - profitPips
      if (giveback >= inst.trailGivebackPips) { … await closePosition(p.ticket) }
    }
  }
}
```
- 漁師: `targetTouchTP:false`＝(B)を取らず(D)天井トレイル維持／`stopLoss`＝`row.stopLoss`(予想SL価格)到達で(A)能動カット。
- WAX: `targetTouchTP:true`＝(B)目標(takeProfit)タッチで確定／`stopLoss`＝M5実体S/R±8p突破&ヒゲ判定で(A)。
- 共通: (C)建値バックストップ(peak≥8p→建値圏)・(D)天井トレイル。

---

## 3. サーバーサイド・ブラケット注文の現状の配線実態

### 3.1 データフロー（openPosition → MT5サーバー Python → order_send の sl/tp）

**(1) backend `mt5trade.ts:107` openPosition（slPips/tpPips を /order に送る）**
```ts
export async function openPosition(
  direction: Direction, lots: number, slPips?: number, tpPips?: number, agent?: string, comment?: string,
): Promise<MT5Order> {
  return mt5Post<MT5Order>('/order', {
    direction, lots,
    sl_pips: slPips,
    tp_pips: tpPips,
    agent:   agent ?? '',
    comment: comment ?? '',
  })
}
```

**(2) MT5サーバー Python `mt5server/mt5_server.py` `/order`→`_market_order`（sl/tp価格を計算し order_send に載せる＝サーバーサイド・ブラケット）**
```python
def _market_order(direction, lots, sl_pips, tp_pips, comment):
    …
    if sl_pips is not None and sl_pips > 0:
        sl = round(price - sl_pips * PIP_SIZE, 3) if is_buy else round(price + sl_pips * PIP_SIZE, 3)
    if tp_pips is not None and tp_pips > 0:
        tp = round(price + tp_pips * PIP_SIZE, 3) if is_buy else round(price - tp_pips * PIP_SIZE, 3)
    req = {
        "action":  mt5.TRADE_ACTION_DEAL, "symbol": SYMBOL, "volume": float(lots),
        "type":    mt5.ORDER_TYPE_BUY if is_buy else mt5.ORDER_TYPE_SELL,
        "price":   price,
        "sl":      sl,          # ← サーバーサイドSL
        "tp":      tp,          # ← サーバーサイドTP
        "deviation": DEVIATION_POINTS, "magic": 20260515,
        "type_time": mt5.ORDER_TIME_GTC, "type_filling": mt5.ORDER_FILLING_IOC,
    }
    result = mt5.order_send(req)
```

**(3) 漁師の発注（`ai/fishermanTrader.ts:294-296`・slPips/tpPipsは「予想SL/TP価格」由来）**
```ts
const slPips = eff.sl != null ? clamp((eff.direction === 'buy' ? (price - eff.sl) : (eff.sl - price)) / PIP, MIN_SL_PIPS, MAX_HARD_SL_PIPS) : MAX_HARD_SL_PIPS
const tpPips = eff.tp != null ? clamp((eff.direction === 'buy' ? (eff.tp - price) : (price - eff.tp)) / PIP, MIN_TP_PIPS, MAX_TP_PIPS) : MAX_TP_PIPS
const order = await ctx.openPosition(eff.direction, a.lots, slPips, tpPips, FISHERMAN_TAG, `fisherman fc#${eff.fcId}`)
```

→ 結論（事実）: **ブラケット注文の配線は backend→Python→order_send まで全て存在し、漁師も発注時に `sl_pips`/`tp_pips` を渡している**（MT5側にSL/TPが置かれる）。

### 3.2 なぜ漁師でブラケットが「有効に機能せず、遅すぎる能動決済に頼る」結果になっていたか（ロジック上の接続問題）

事実関係のみ:

1. **TPが「AIの予想takeProfit」由来で、スキャルの実効目標になっていない。** `fishermanTrader.ts:294` の `tpPips` は `eff.tp`（forecastの予想TP＝対向S/R手前等で遠いことが多い）から算出。`fishInstance` は `certainExit.targetTouchTP:false`（`fishermanTrader.ts:46`）＝**目標タッチで取らず天井トレイル維持**。よってブローカーTPは「遠い保険」で、実際の利確は能動層に委ねられる。

2. **利益確定が能動2層（comms依存）に依存している。** §2の通り、漁師の実利確は
   - `activeExit.manageActiveExits`（(C)建値バックストップ・(D)天井トレイル・(A)予想SL能動カット）＝`fishermanTrader.ts:242`
   - `exitGuardian.exitTick`（精密トレイル/棚到達/波死亡/180s収縮/AI船長）＝`autotrader.ts:1221`
   の両方が `await closePosition(ticket)`（backend→MT5 `/close`）で執行する。**この `/close` 実行が MT5端末の認証断・タイムアウトで失敗すると、利確が執行されない。**

3. **実測される失敗（本番 `~/.pm2/logs/fx-backend-error.log`・生ログ）:**
   ```
   [autotrader] close detection error: Error: MT5 /positions HTTP 503: mt5.initialize() failed: (-6, 'Terminal: Authorization failed')
   [autotrader] close detection error: TimeoutError: The operation was aborted due to timeout
   ```
   `/positions` が 503（端末認証失敗）の間、建玉照会が返らず、能動決済（trail/棚/波死亡/予想SL）の判定・執行が成立しない。

4. **接続上の問題点（ロジック）:** ブローカー側に置かれている `sl_pips/tp_pips` は「遠い保険SL／遠い予想TP」であり、**スキャルの利確を取りに行く主体は backend 駆動の能動2層**になっている。したがって、注文時にブローカーへ預けたブラケット（端末断でも自動執行される）が「実効的な利確の主決済」になっておらず、利確の成否が backend↔MT5 の通信状態に依存している。＝「真っ当な固定pipsをブローカーTPで確実に取る」設計には、(a) 漁師のTP/SLを実効スキャル目標の固定値にする、(b) 能動2層を漁師から切り離す、の2点が未実装。

---

（本書は読み取りのみで生成。凍結5は無改変。各コードは生成時点の `backend/src/` 実体に基づく。）
