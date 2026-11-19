# Qindee ቂንዴ — architecture of the FINAL prototype (`index.html`)

Single static file (HTML + CSS + vanilla JS, no build step, no framework). Deploys as-is on Vercel/GitHub Pages.
Only external script: SheetJS (cdnjs, pinned 0.18.5) for Excel import. Logo emblem is an inline base64 PNG.

## 1. State (one JSON object `S`, saved to `localStorage['suke3']`; documents in `localStorage['suke3d']`)
| Key | Shape |
|---|---|
| `users[]` | `{id,name,ph,role:'owner'|'manager'|'cashier',pin(=password),on:1|0,q1,a1,q2,a2}` |
| `items[]` | `{id,n,u:'piece'|'kg'|'litre',cost,price,qty,re(low-stock alert),e(expiry YYYY-MM-DD),b(batch),bc(barcode)}` |
| `sales[]` | `{id,no,t(ms),u(userId),late,co(cash given back),cust,tot,lines[{i,n,q,p,o(original price),c(cost),u}],pay{cash,mob,cr,bank}}` — `pay.cash` is NET (can be negative after cash given back); `cash+mob+cr = tot` |
| `voids[]` | a sale object + `void{t,u,r(reason),by(approver)}` (voided sales are moved out of `sales`) |
| `cp[]` | credit payments `{id,sid(sale id),t,amt,m('Cash'|bank),u}` |
| `exp[]` | expenses `{id,n,amt,d,rem?}` |
| `sup[]` | supplier records `{id,n,d,tot,paid}` |
| `rem[]` / `rh[]` | reminders `{id,t,amt,due,rep:'none|monthly|quarterly|yearly',bf(days before),uid?}` / history `{id,t,amt,date,x:'added'|'skipped'|'stopped'}` |
| `adj[]` | stock adjustments `{id,i,n,d(±qty),r(reason),c(cost),u,t}` |
| `log[]` | `{name,a:'in'|'out',t}` |
| `shop` | `{name,plan,trialEnd,subEnd,pays[{id,plan,name,amt,days,m,ref,fn,t,st:'pending'|'approved'}]}` |
| `set` | `{dc,dm(max discount % cashier/manager),rw(receipt width 58|80),rf(receipt footer)}` |
| misc | `base`(all-time sales start), `nextNo`(receipt no.), `lang`('en'|'am'|'om') |
| `DOC[]` (separate key) | uploaded documents `{id,t,c,d,amt,m(mime),u(dataURL)}` |

## 2. Code layout (top to bottom of the `<script>`)
1. `D` translation dictionary (en→am/om) + `T()`; DOM auto-translator (`trN`, MutationObserver) for leftover English.
2. Helpers: `day, ds (YYYY-MM-DD), money, priv (hide-money), tm, cls, dl, minE, un, cpd, save`.
3. `seed()` demo data → `S`; `agg(a,b)` = report aggregator (sales, cost, cash, mobile, credit, cash-given, credit collected, discounts); `drawer()` = cash in drawer.
4. Shell: `render()` (login or app shell), `go(view)`, `views{}` (home, sell, mine, stock, sup, cred, rep, act, staff + admin sub-tabs rem/doc/staff/act/set/sub), `modal()`, `notice()`, `page()` (search list helper), `W()`.
5. Sell: `add, qty, setQ, setB, ed (discount modal), pm, cartV, done` (+ `wq` unused), `voidS`, `adjS`, `cpay`, `spay`, `remE/remP/remD`, `imp` (Excel import), `repH` (report HTML), `pdf` (print overlay).
6. Patch layers, each appended as a block (`/* ===== Qindee vN ===== */`):
   - v6 phone sign-in (`login, forgot, reg, nph, okph`), subscription (`SH, PLANS, PAYINFO, endT, dleft, locked, subV, payS, appr, setDays`), banner, new import
   - v7 Ethiopian calendar (`etd, ets`), phone-input limiter, WhatsApp/CBE payment info, shop name on PDF
   - v8 compact phone nav, v9 header + SVG icons (`IC`), non-negative money guards (`_modal` wrapper), v10 PC header + PC two-column Sell, v11 split-screen login, v12 receipts (`rcpt`, `rtxt`, 58/80 mm print, share)
   > IMPORTANT: many functions are declared more than once and wrapped (`const _x=fn; fn=function(){…_x()…}`). **The last declaration wins.** When porting, consolidate to one definition per function.

## 3. Business rules to keep (port exactly)
- Stock never < 0; money fields ≥ 0 (stock adjustment may be negative); cost, price, qty, low-stock alert > 0 on items.
- Expiry (optional) must be > today + 90 days for new items and restocks.
- Weighed items (kg/litre): tap adds 1; ¼ ½ ¾ 1 2, free quantity, or "by birr"; line total rounded to whole birr (`lt`).
- Payments: cash | mobile (bank required) | mixed (cash received + transferred; cash to give back = cash + transfer − total) | credit (customer name required). Cash-exchange sale = empty cart + mobile.
- Discount: % ⇄ birr ⇄ new price linked; cashier limit `set.dc`, manager `set.dm` (default 10 / 30); above limit needs owner password.
- Void: cashier only own sale ≤ 10 min, otherwise owner/manager password; credit sale with payments can't be voided; stock returns; reports exclude voids.
- Credit paid by cashier allowed (recorded with collector); owner/manager may reverse.
- Subscription: 30-day trial → plans monthly 2,000 / 3 mo 5,000 / 6 mo 8,999 birr; banner at ≤ 7 days; 5 grace days; then selling locked (reports stay readable). Receipt picture OR reference required.
- Roles: owner (all), manager (no Admin), cashier (Sell, My sales, Items w/o cost, Credit).

## 4. Mapping to Supabase (suggested)
`shops`, `profiles(id→auth.users, shop_id, role, name, phone, active)`, `items`, `sales`, `sale_lines`, `sale_voids`, `credit_payments`, `expenses`, `supplier_records`, `reminders`, `reminder_history`, `stock_adjustments`, `activity_log`, `payments` (subscription receipts), `documents` (+ Storage bucket), `daily_summary`.
Every table has `shop_id`; RLS: `shop_id = (select shop_id from profiles where id = auth.uid())`, plus role checks (cashier cannot read cost/profit, cannot void others' sales) and a subscription check on writes. Auth: phone → hidden email (`0911223344@qindee.app`) + password ≥ 6 chars (then PIN/fingerprint on device). Stock changes via an RPC (`qty = qty ± n` in one transaction). Admin approval of payments via service-role/edge function only.
