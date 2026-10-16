// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
//
// Minervini VCP Detector
// Stage 2 (Trend Template) + contraction sequence + volume dry-up + pivot / breakout.
// Designed for DAILY charts. Swing points are mechanical (ta.pivothigh / ta.pivotlow)
// and are confirmed `pivot_len` bars late, so they never repaint but also never lead.

//@version=6
indicator('Minervini VCP Detector', shorttitle = 'VCP', overlay = true, max_bars_back = 500)

// ==========================================
// 1. INPUTS
// ==========================================
grp_disp = '1. Display'
table_pos_input = input.string('Top Right', 'Table Position', options = ['Top Left', 'Top Center', 'Top Right', 'Middle Left', 'Middle Center', 'Middle Right', 'Bottom Left', 'Bottom Center', 'Bottom Right'], group = grp_disp)
table_size_input = input.string('Small', 'Table Size', options = ['Tiny', 'Small', 'Normal', 'Large', 'Huge'], group = grp_disp)
show_lines = input.bool(true, 'Plot Pivot / Stop Lines', group = grp_disp)

grp_ma = '2. Moving Averages'
ma50_len = input.int(50, '50-Day MA Length', minval = 1, group = grp_ma)
ma150_len = input.int(150, '150-Day MA Length', minval = 1, group = grp_ma)
ma200_len = input.int(200, '200-Day MA Length', minval = 1, group = grp_ma)
c_ma50 = input.color(color.blue, '50-Day Color', group = grp_ma)
c_ma150 = input.color(color.orange, '150-Day Color', group = grp_ma)
c_ma200 = input.color(color.red, '200-Day Color', group = grp_ma)

grp_tt = '3. Trend Template (Stage 2 gate)'
ma200_slope_bars = input.int(84, 'Criterion 3: 200MA Rising For (bars)', minval = 20, group = grp_tt, tooltip = 'Minervini: at least 1 month (~20 bars), preferably 4-5 months (~84-105 bars).')
low_above_pct = input.float(25, 'Criterion 6: Min. % Above 52-Week Low', minval = 0, step = 5, group = grp_tt)
high_within_pct = input.float(25, 'Criterion 7: Max. % Below 52-Week High', minval = 0, step = 5, group = grp_tt)
use_rs_tt = input.bool(true, 'Criterion 8: Require RS Proxy', group = grp_tt, tooltip = 'Uses a benchmark-relative PROXY, not the IBD RS rating. Disable to gate on criteria 1-7 only.')
rs_min = input.float(70, 'Criterion 8: Min. RS Proxy', minval = 1, maxval = 99, step = 5, group = grp_tt)
bm_sym = input.symbol('SPY', 'Benchmark Symbol (RS proxy)', group = grp_tt)

grp_vcp = '4. VCP Structure'
pivot_len = input.int(5, 'Swing Pivot Strength (bars each side)', minval = 2, maxval = 15, group = grp_vcp, tooltip = 'Bars on each side required to confirm a swing high/low. Larger = fewer, more significant swings, but later confirmation.')
max_base_bars = input.int(150, 'Max Base Length (bars)', minval = 30, maxval = 400, group = grp_vcp)
min_base_bars = input.int(15, 'Min Base Length (bars)', minval = 5, group = grp_vcp, tooltip = 'Minervini: base of at least ~3 weeks.')
min_contr = input.int(2, 'Min. Contractions', minval = 2, maxval = 6, group = grp_vcp)
max_first_depth = input.float(35, 'Max. Depth of First Contraction %', minval = 5, step = 5, group = grp_vcp)
max_last_depth = input.float(10, 'Max. Depth of Final Contraction %', minval = 1, step = 1, group = grp_vcp)
shrink_pct = input.float(100, 'Each Contraction <= X% of Previous', minval = 10, maxval = 100, step = 5, group = grp_vcp, tooltip = '100 = merely smaller or equal. 70 = each pullback must be at most 70% as deep as the one before.')
rising_lows = input.bool(true, 'Require Rising Lows', group = grp_vcp)
low_tol = input.float(2, 'Rising-Low Undercut Tolerance %', minval = 0, step = 0.5, group = grp_vcp)

grp_entry = '5. Pivot, Volume & Risk'
near_pct = input.float(5, 'READY: Within X% Below Pivot', minval = 0.5, step = 0.5, group = grp_entry)
buy_zone = input.float(5, 'Buy Zone: Max. % Above Pivot', minval = 0.5, step = 0.5, group = grp_entry, tooltip = 'Minervini: do not chase more than ~5% above the pivot.')
vol_short_len = input.int(10, 'Recent Volume Avg (bars)', minval = 3, group = grp_entry)
vol_long_len = input.int(50, 'Baseline Volume Avg (bars)', minval = 10, group = grp_entry)
dry_ratio = input.float(80, 'Dry-Up: Recent <= X% of Baseline', minval = 10, maxval = 100, step = 5, group = grp_entry)
brk_mult = input.float(1.4, 'Breakout Volume >= X x Baseline', minval = 1, step = 0.1, group = grp_entry)
req_dry = input.bool(true, 'READY Requires Volume Dry-Up', group = grp_entry)
req_tight = input.bool(true, 'READY Requires Tight Closes', group = grp_entry)
tc_len = input.int(5, 'Tight Closes Lookback (bars)', minval = 3, group = grp_entry)
tc_mult = input.float(1.5, 'Tight Closes: Close Range < X x ATR(14)', minval = 0.3, step = 0.1, group = grp_entry)
max_risk = input.float(8, 'Max. Risk % (Pivot to Final Low)', minval = 1, step = 0.5, group = grp_entry)

grp_col = '6. Colors'
c_text_lbl = input.color(color.white, 'Label Text', group = grp_col)
c_text_val = input.color(color.white, 'Value Text', group = grp_col)
c_bg_lbl = input.color(color.new(color.gray, 20), 'Label Background', group = grp_col)
c_good = input.color(color.green, 'GOOD', group = grp_col)
c_warn = input.color(color.yellow, 'WARNING', group = grp_col)
c_bad = input.color(color.new(color.red, 10), 'BAD', group = grp_col)
c_neutral = input.color(color.gray, 'NEUTRAL', group = grp_col)
c_tight = input.color(color.aqua, 'READY / TIGHT', group = grp_col)

// ==========================================
// 2. STAGE 2 GATE (Minervini Trend Template)
// ==========================================
ma50 = ta.sma(close, ma50_len)
ma150 = ta.sma(close, ma150_len)
ma200 = ta.sma(close, ma200_len)
plot(ma50, title = '50 SMA', color = c_ma50, linewidth = 2)
plot(ma150, title = '150 SMA', color = c_ma150, linewidth = 2)
plot(ma200, title = '200 SMA', color = c_ma200, linewidth = 2)

roc(src, len) =>
    (src - src[len]) / src[len] * 100

bm_close = request.security(bm_sym, timeframe.period, close, ignore_invalid_symbol = true)
rs_raw = roc(close, 63) * 0.4 + roc(close, 126) * 0.2 + roc(close, 189) * 0.2 + roc(close, 252) * 0.2
bm_raw = roc(bm_close, 63) * 0.4 + roc(bm_close, 126) * 0.2 + roc(bm_close, 189) * 0.2 + roc(bm_close, 252) * 0.2
rpr_calc = 50 + rs_raw - bm_raw
rpr_proxy = na(rpr_calc) ? na : math.max(1, math.min(99, rpr_calc))
rpr_txt = na(rpr_proxy) ? 'n/a' : str.tostring(rpr_proxy, '#.0')
rpr_col = na(rpr_proxy) ? c_neutral : rpr_proxy > 80 ? c_good : rpr_proxy >= rs_min ? c_warn : c_neutral

high_52wk = ta.highest(high, 252)
low_52wk = ta.lowest(low, 252)

c1 = close > ma150 and close > ma200
c2 = ma150 > ma200
c3 = ma200 > ma200[ma200_slope_bars]
c4 = ma50 > ma150 and ma50 > ma200
c5 = close > ma50
c6 = close >= low_52wk * (1 + low_above_pct / 100)
c7 = close >= high_52wk * (1 - high_within_pct / 100)
c8 = not na(rpr_proxy) and rpr_proxy >= rs_min

stage2 = c1 and c2 and c3 and c4 and c5 and c6 and c7 and (use_rs_tt ? c8 : true)
tt_score = (c1 ? 1 : 0) + (c2 ? 1 : 0) + (c3 ? 1 : 0) + (c4 ? 1 : 0) + (c5 ? 1 : 0) + (c6 ? 1 : 0) + (c7 ? 1 : 0) + (use_rs_tt and c8 ? 1 : 0)
tt_max = use_rs_tt ? 8 : 7
tt_txt = stage2 ? 'PASSED' : 'WAIT ' + str.tostring(tt_score) + '/' + str.tostring(tt_max)

// ==========================================
// 3. SWING PIVOTS (alternating high / low store)
// ==========================================
var array<float> pv_price = array.new<float>()
var array<int> pv_bar = array.new<int>()
var array<int> pv_type = array.new<int>()  // 1 = swing high, -1 = swing low

f_add_pivot(array<float> pp, array<int> pb, array<int> pt, float p, int b, int t) =>
    int n = array.size(pt)
    if n > 0 and array.get(pt, n - 1) == t
        // two consecutive pivots of the same type: keep the more extreme one
        float last = array.get(pp, n - 1)
        if t == 1 and p > last or t == -1 and p < last
            array.set(pp, n - 1, p)
            array.set(pb, n - 1, b)
    else
        array.push(pp, p)
        array.push(pb, b)
        array.push(pt, t)
        if array.size(pt) > 40
            array.shift(pp)
            array.shift(pb)
            array.shift(pt)
    true

ph = ta.pivothigh(high, pivot_len, pivot_len)
pl = ta.pivotlow(low, pivot_len, pivot_len)

// Mutate only on confirmed bars so realtime ticks cannot corrupt the store.
if barstate.isconfirmed
    if not na(ph)
        f_add_pivot(pv_price, pv_bar, pv_type, ph, bar_index - pivot_len, 1)
    if not na(pl)
        f_add_pivot(pv_price, pv_bar, pv_type, pl, bar_index - pivot_len, -1)

// ==========================================
// 4. CONTRACTION CHAIN
// ==========================================
// Walks backward from the latest completed high->low pullback. A pullback is kept only if
// the one after it is no deeper (x shrink ratio) and, optionally, its low is not lower.
// Returns: [count, first depth %, last depth %, pivot, final low, final-low bar, base start bar, text]
f_vcp(array<float> pp, array<int> pb, array<int> pt, int cur, int maxBase, float shrink, bool risingLows, float lowTol) =>
    int cnt = 0
    float firstD = na
    float lastD = na
    float finalLow = na
    int finalHiIdx = -1
    int finalLowBar = na
    int startBar = na
    string txt = ''
    float laterDepth = na
    float laterLow = na
    int k = array.size(pt) - 1
    while k >= 1
        if array.get(pt, k) == -1 and array.get(pt, k - 1) == 1
            float h = array.get(pp, k - 1)
            float l = array.get(pp, k)
            int hb = array.get(pb, k - 1)
            float d = (h - l) / h * 100
            bool ok = hb >= cur - maxBase
            if ok and cnt > 0
                ok := laterDepth <= d * shrink and (not risingLows or laterLow >= l * (1 - lowTol / 100))
            if not ok
                break
            if cnt == 0
                lastD := d
                finalLow := l
                finalHiIdx := k - 1
                finalLowBar := array.get(pb, k)
            firstD := d
            startBar := hb
            txt := str.tostring(d, '#.#') + '%' + (cnt > 0 ? ' → ' + txt : '')
            laterDepth := d
            laterLow := l
            cnt += 1
            k -= 2
        else
            k -= 1
    // Pivot = highest swing high from the final contraction's high onward (the base's resistance)
    float pivot = na
    if cnt > 0
        for j = finalHiIdx to array.size(pt) - 1
            if array.get(pt, j) == 1
                float p = array.get(pp, j)
                if na(pivot) or p > pivot
                    pivot := p
    [cnt, firstD, lastD, pivot, finalLow, finalLowBar, startBar, txt]

[n_c, first_d, last_d, pivot, final_low, flow_bar, start_bar, chain_txt] = f_vcp(pv_price, pv_bar, pv_type, bar_index, max_base_bars, shrink_pct / 100, rising_lows, low_tol)

// Lowest low since the final contraction low (detects an undercut that is not yet a confirmed pivot)
float cur_low = na
if n_c > 0
    cur_low := low
    for i = 1 to math.min(bar_index - flow_bar, 400)
        cur_low := math.min(cur_low, low[i])

base_bars = na(start_bar) ? 0 : bar_index - start_bar
shape_ok = n_c >= min_contr and first_d <= max_first_depth and last_d <= max_last_depth and base_bars >= min_base_bars and not (cur_low < final_low)

// ==========================================
// 5. VOLUME, TIGHT CLOSES, SIGNALS
// ==========================================
vol_s = ta.sma(volume, vol_short_len)
vol_l = ta.sma(volume, vol_long_len)
vol_ratio = na(vol_l) or vol_l == 0 ? na : vol_s / vol_l * 100
dry_up = not na(vol_ratio) and vol_ratio <= dry_ratio
brk_vol = not na(vol_l) and vol_l > 0 and volume >= vol_l * brk_mult

atr14 = ta.atr(14)
tc_range = ta.highest(close, tc_len) - ta.lowest(close, tc_len)
tight_close = not na(atr14) and tc_range < tc_mult * atr14

pivot_ok = not na(pivot)
above_piv = pivot_ok and close > pivot
in_zone = above_piv and close <= pivot * (1 + buy_zone / 100)
extended = above_piv and not in_zone
dist_pivot = pivot_ok ? (pivot - close) / pivot * 100 : na
near_pivot = pivot_ok and not above_piv and dist_pivot <= near_pct

setup = stage2 and shape_ok
ready = setup and near_pivot and (not req_dry or dry_up) and (not req_tight or tight_close)
breakout = setup and in_zone and brk_vol and close[1] <= pivot

risk_pct = pivot_ok and not na(final_low) ? (pivot - final_low) / pivot * 100 : na

status_txt = not stage2 ? 'No Stage 2' : not shape_ok ? 'No VCP' : breakout ? 'BREAKOUT' : extended ? 'Extended' : in_zone ? 'Above pivot (weak vol)' : ready ? 'READY' : 'Forming'
status_col = not stage2 ? c_neutral : not shape_ok ? c_neutral : breakout ? c_good : extended ? c_bad : in_zone ? c_warn : ready ? c_tight : c_warn

up_vol = math.sum(close > close[1] ? volume : 0, 20)
dn_vol = math.sum(close < close[1] ? volume : 0, 20)
pressure_txt = up_vol > dn_vol ? 'Up-day vol dominant' : up_vol < dn_vol ? 'Down-day vol dominant' : 'Balanced'
pressure_col = up_vol > dn_vol ? c_good : up_vol < dn_vol ? c_bad : c_neutral

// ==========================================
// 6. PLOTS
// ==========================================
plot(show_lines and setup ? pivot : na, title = 'VCP Pivot', color = color.lime, style = plot.style_linebr, linewidth = 2)
plot(show_lines and setup ? final_low : na, title = 'VCP Stop (final low)', color = color.red, style = plot.style_linebr, linewidth = 1)
plotshape(ready and not ready[1], title = 'READY', style = shape.circle, location = location.belowbar, color = c_tight, size = size.tiny)
plotshape(breakout, title = 'BREAKOUT', style = shape.triangleup, location = location.belowbar, color = c_good, size = size.small, text = 'BO')

// ==========================================
// 7. TABLE
// ==========================================
pos = switch table_pos_input
    'Top Left' => position.top_left
    'Top Center' => position.top_center
    'Top Right' => position.top_right
    'Middle Left' => position.middle_left
    'Middle Center' => position.middle_center
    'Middle Right' => position.middle_right
    'Bottom Left' => position.bottom_left
    'Bottom Center' => position.bottom_center
    => position.bottom_right

sz = switch table_size_input
    'Tiny' => size.tiny
    'Small' => size.small
    'Normal' => size.normal
    'Large' => size.large
    => size.huge

f_row(table t, int r, string lbl, string val, color bg) =>
    table.cell(t, 0, r, lbl, bgcolor = c_bg_lbl, text_color = c_text_lbl, text_size = sz)
    table.cell(t, 1, r, val, bgcolor = bg, text_color = c_text_val, text_size = sz)
    true

var tbl = table.new(pos, 2, 10, border_width = 1)

if barstate.islast
    int row = 0
    if not timeframe.isdaily
        f_row(tbl, row, 'Timeframe', 'Use Daily!', c_bad)
        row += 1

    f_row(tbl, row, 'Trend Template', tt_txt, stage2 ? c_good : c_bad)
    row += 1
    f_row(tbl, row, 'RS proxy vs ' + bm_sym, rpr_txt, rpr_col)
    row += 1
    f_row(tbl, row, 'VCP Setup', status_txt, status_col)
    row += 1
    f_row(tbl, row, 'Contractions', n_c > 0 ? str.tostring(n_c) + 'x: ' + chain_txt : 'none', shape_ok ? c_good : c_neutral)
    row += 1
    f_row(tbl, row, 'Vol dry-up (' + str.tostring(vol_short_len) + '/' + str.tostring(vol_long_len) + ')', na(vol_ratio) ? 'n/a' : str.tostring(vol_ratio, '#') + '%' + (dry_up ? ' DRY' : ''), dry_up ? c_good : c_neutral)
    row += 1
    f_row(tbl, row, 'Tight closes', tight_close ? 'TIGHT' : 'Normal', tight_close ? c_tight : c_neutral)
    row += 1
    f_row(tbl, row, 'Pivot', setup and pivot_ok ? str.tostring(pivot, format.mintick) + (dist_pivot >= 0 ? '  (' + str.tostring(dist_pivot, '#.0') + '% below)' : '  (' + str.tostring(-dist_pivot, '#.0') + '% above)') : 'n/a', setup ? c_good : c_neutral)
    row += 1
    f_row(tbl, row, 'Risk (pivot→stop)', setup and not na(risk_pct) ? str.tostring(risk_pct, '#.0') + '%' : 'n/a', not setup or na(risk_pct) ? c_neutral : risk_pct <= max_risk ? c_good : c_bad)
    row += 1
    f_row(tbl, row, 'Pressure (20d)', pressure_txt, pressure_col)

// ==========================================
// 8. ALERTS (edge-triggered)
// ==========================================
alertcondition(breakout, title = 'VCP Breakout', message = 'VCP breakout above pivot on {{ticker}} ({{interval}})')
alertcondition(ready and not ready[1], title = 'VCP Ready', message = 'VCP setup near pivot on {{ticker}} ({{interval}})')
alertcondition(stage2 and not stage2[1], title = 'Trend Template Newly Passed', message = 'Trend Template newly passed on {{ticker}} ({{interval}})')
