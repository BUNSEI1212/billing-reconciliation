# ledger-recon — 突合レポート / Reconciliation Diff Report

**日本の中小事業者（介護請求業務）向け 月次突合レポートツール**
**Monthly reconciliation diff report for Japanese SME billing (long-term care / kaigo billing)**
**日本中小企业（介护请求业务）的月度对账差异报告工具**

[日本語](#日本語) · [English](#english) · [中文](#中文)

---

## 日本語

### これは何か

PDF を読みません。OCR も行いません。お客様の入力を触りません。
答えるのは一つの問いだけです：

> **今月の請求、どこが合っていないか。何行目か。前回確認済みか。**

3 つの表を入力すると、Excel のレポートを 1 枚出力します。
1 行目は「今月 N 件の不一致、うち M 件は要確認」。
以下、1 行ごとに「何行目・誰の・どのサービス・請求額はいくら・契約単価はいくら」を示します。

### なぜ作ったか

書類の読み取り（OCR と構造化）は、すでに安価な標準機能です。
無料・低価格のサービスが普及し、使うほど精度も上がります。
ただしそれらは**書類をデータにする**ところまでで、
**お客様ごとの業務ルールに照らして突合する**ことは行いません。

突合の工程は、法人ごとにルールが異なります。
契約単価がいつ改定されたか、今月はそもそも計上しない項目はどれか、
前回なぜ手作業で見送ったのか。こうした個別事情を、
パッケージ製品が 1 法人のために作り込むことはありません。人が担うしかありません。

本ツールの位置づけはここです：**読み取りとは競合せず、突合の層だけを担う。**
さらに**前回の判断を覚える**こと。これが手作業との決定的な差です。
人は先月どう判断したかを覚えていられません。

### 出力イメージ

Excel 1 枚（`out/演示用_突合レポート_2026-10.xlsx`、PNG プレビュー付き）：

```
突合レポート　デモ用（模擬データ）　対象期間 2026-10
※模擬データです。実在の法人・利用者・会計データは一切含まれていません。
今月 5 件の不一致、うち 5 件は要確認（…）

#  差異種別              該当箇所                        該当行の値      推奨修正値 / 判定理由
1  単価と契約の不一致      6 行目 鈴木 次郎 / 訪問介護…     単価 1,300     契約単価 1,250（金額 6,250）
2  契約にない項目          9 行目 伊藤 五郎 / 緊急訪問介護…  単価 5,000     契約単価表に該当なし
3  余分な行               10 行目 渡辺 六郎 / 通院介助      数量 2 / 金額 4,000  今月の計画に含まれていません
4  契約開始より前の日付     11 行目 中村 七子 / 訪問介護…     2026-01-20   【契約未カバー】契約は 2026-04-01 から
5  金額の不一致           14 行目 木村 十郎 / 訪問介護…     金額 9,500    数量×単価 = 9,150（差 -350）
合計 5 件
```

### 入力：3 つの表

| 表 | 必須列 | 内容 |
|---|---|---|
| 利用実績 | 必須：利用者名 / 品目 / 数量 / 単価 / 金額<br>任意：サービス日 | 当月に実際に計上された明細 |
| 契約単価表 | 必須：利用者名 / 品目 / 契約単価<br>任意：適用開始日 | 適用開始日による分段が可能（改定は日常的） |
| 計画明細 | 必須：利用者名 / 品目 | **この表がないと、未計上・余分な行の判定は行いません** |

**表が欠けた場合の挙動：**

- **契約単価表**がない → 全行が「契約にない項目」として報告される（誤報に見えます）
- **計画明細**がない → 未計上・余分な行の 2 種類を無効化。8 種のうち 7 種のみ
- **利用実績**がない → 実行できません

突合とは 2 つのものを突き合わせる行為です。**片方しかなければ、突合にはなりません。**

**任意列を省略した場合：** `サービス日` がないと日付系の検査（対象期間外・契約開始前）が無効になります。`適用開始日` がないと分段が使えず、同一品目に複数行があるとエラーになります。

**列名は日本語のまま変更しないでください。** コードは `利用者名` / `品目` / `数量` / `単価` / `金額` / `サービス日` / `契約単価` / `適用開始日` を文字列で完全一致させています。英語等に置き換えると読み込みに失敗します。

### 差異の 8 種別

| # | 種別 | 位置精度 | 状態 |
|---|---|---|---|
| 1 | 単価と契約の不一致 | 行単位 | ✅ サービス日で有効な分段を選択 |
| 2 | 金額の不一致 | 行単位 | ✅ 種別 1 と並列出力（統合しない） |
| 3 | 未計上（計画出力漏れ） | 行単位 | ✅ 計画表のみを基準に判定 |
| 4 | 余分な行 | 行単位 | ✅ 理由を必ず明示 |
| 5 | 消費税額の誤り | 合計単位 | ⬜ **利用不可・税率区分が未確定** |
| 6 | 対象期間外の日付 | 行単位 | ✅ |
| 7 | 契約にない項目 | 行単位 | ✅ ルール台帳に記憶可能 |
| 8 | 日付が読み取れない | 行単位 | ✅ 別段落、件数に不算入 |

「サービス日が契約の全分段より前」は第 9 種別を新設せず、第 7 種別に統合し、
文言の子理由 `【契約未カバー…】` で区別します。
お客様から見ればどちらも「この費用には契約根拠がない」であり、
種別を増やすと覚える概念が増えるためです。

### 中核の仕組み：ルール台帳

人による確認は**追記のみ**。上書きはしません：

```
{顧客, 差異種別, 内容の指紋, 判定, 適用月, 確認時の金額}
```

翌月、同種の差異が命中した際に、三档の閾値で再確認の要否を決めます：

```
変動 > 10,000 円                   → 要確認に戻す
上記以外で 変動率 > 20% かつ > 1,000 円 → 要確認に戻す
それ以外                            → 通知しない（「確認済み・変更不要」）
```

**単純な割合基準にしない理由：** 183,000 → 150,000 は 18% 減で、
割合だけなら見逃します。しかし実額では 33,000 円の不足であり、
レポートに「今月は問題なし」と書くことになります。三档はこれを防ぎます。
逆に 500 → 620 は 24% 増でも差額 120 円であり、財務上わざわざ知らせる価値はありません。

**指紋には数値の文脈が必須です。** 以前の実装は指紋が `(顧客, 品目)` のみで、
「鈴木の単価 1,300 を特例として承認」した後、翌月に誤って 1,500 と入力されても
ルールが通過させてしまう状態でした。
現在の指紋は `unitprice|鈴木 次郎|訪問介護（生活援助2)|1,300|1,250` のような形式です。

### 判定状態（三状態）

| 状態 | 表示文言 | 意味 |
|---|---|---|
| `NEW` | 要確認 | 初出、または数値が履歴と異なる |
| `CONFIRMED_SAME` | 確認済み・変更不要 | 同一の数値、前回の判定を踏襲 |
| `RECURRING` | 前回修正済み・今回再発 | 前回修正した同じ誤りが再び出現 |

第四状態 `CONFIRMED_DIVERGED` は存在しましたが**削除しました**。
全フローを通して一度も出力されなかったためです。
テスト内でしか現れず、業務で決して現れない状態を対外資料に書くことは、
お客様に対して使えない約束を提示するのと同じです。

### 既知の限界（重要）

**できていないことを先に読んでください。導入可否の判断はその後にしてください：**

1. **実在の日本の請求書を一切見ていません。** 正規化も各種判定も、自作データで検証したものです。
   実際の書類はさらに崩れます：印影の重なり、手書き金額、ページ跨ぎの表。
2. **原文の指摘は「N 行目」までで、「N ページ M 行目を枠で囲む」ことはできません。**
   座標特定には実際の PDF レイアウトの取り込みが必要です。
3. **消費税額の検証は利用できません。** 介護報酬では利用者負担と保険給付で税率が異なり、
   行単位の税率区分が未確定です。誤った税額を出すより、出さない方が害が小さいと判断しました。
4. **未計上の金額は契約単価から推計しています。** 契約改定を「変動あり」と誤判定し、
   余計な再確認を求める場合があります。
5. **`final_check.py` の一部のケースは初期の自前テストから移したもの**で、
   期待値を 1 件ずつ見直してはいません。問題発生時はまずそこを疑ってください。
   `review_gate.py` の B1〜B4 は一から書き、1 件ずつ確認済みです。

### 実行方法

```bash
python final_check.py     # 受入検査：31 項目、終了コード 0 が合格
python review_gate.py     # 阻害項目 4 件（B1〜B4）
python make_demo.py       # out/ にデモレポートを生成
```

依存：`openpyxl` のみ。**ネットワーク遮断下でも動作します**
（`engine.py` / `rulebook.py` に通信処理はありません）。これは意図的な設計で、
お客様自身の環境で実行できることを優先しています。

**受入の判定基準：** `行目` の出現回数とデータ行の状態数が一致すること（いずれも 6）、
および中国語の残留が 0 であること。この 2 つの数値は同一の差異群を指すため、
一致しなければ凡例や要約行を集計に混入させていることになります。

### 開発記録

このプロジェクト最大の成果はツール自体ではなく、
**「どのような条件で誤った結論を出すか」の一覧**です。
以下はすべて、実在の顧客も利用者もいない段階で自ら発見したものです：

- 顧客が 33,000 円の収入漏れをしている場面で、レポートが「今月は問題なし」と表示した
- 一度の判断を記憶した後、恒久的に通過させ、金額が 100 倍になっても通知しなかった
- `\u` エスケープで検査器に「中国語残留なし」と表示させたが、レポートは中国語のままだった
- レポートの凡例に、決して出現しない判定状態が 1 件載っていた
- 受入スクリプト自身の期待値が逆になっていた

この記録を残す理由は一つです：
**どこが信頼できないかを知っていることは、どこが便利かを知っていることより重要です。**

---

## English

### What this is

It does not read PDFs, does not run OCR, and does not touch the customer's data entry.
It answers exactly one question:

> **Where does this month's billing not add up — which row, and has it been confirmed before?**

Give it three tables and it produces one Excel report.
The first line reads "N discrepancies this month, M of which need human review."
Each row below states: which line, whose, which service item, what the invoice says,
and what the contract says it should be.

### Why it exists

Document recognition (OCR + structuring) is now a cheap commodity. Established services
offer it at low or zero cost, and they improve with use. But they stop at
**turning documents into data** — they do not **reconcile against each customer's own
business rules**.

Reconciliation rules differ per organization: when a contracted unit price was revised,
which items simply should not be billed this month, why a particular line was manually
accepted last time. No packaged product will build those rules for a single customer.
Only a person can.

That is the position this tool takes: **not competing with recognition, working only in
the reconciliation layer.** And critically, it **remembers the customer's previous
judgements** — which is where it beats manual reconciliation. People cannot remember
how they judged the same item last month.

### What the output looks like

One Excel file (`out/演示用_突合レポート_2026-10.xlsx`, with a PNG preview):

```
突合レポート　デモ用（模擬データ）　対象期間 2026-10
※模擬データです。実在の法人・利用者・会計データは一切含まれていません。
今月 5 件の不一致、うち 5 件は要確認（…）

#  Type                  Location                          Row value        Suggested fix / Reason
1  Unit price mismatch    1,300 vs contract 1,250          鈴木 次郎 / 訪問介護…    Contract: 1,250 (total 6,250)
2  Not in contract        5,000, no matching contract row   伊藤 五郎 / 緊急訪問介護…  Not found in contract price table
3  Extra row              2 × unit, 4,000                   渡辺 六郎 / 通院介助    Not in this month's plan
4  Date before contract   2026-01-20, contract starts 04-01  中村 七子 / 訪問介護…    Contract not in effect for this date
5  Amount mismatch        9,500 recorded, 9,150 expected    木村 十郎 / 訪問介護…    5 × 1,830 = 9,150 (diff -350)
Total: 5
```

### Input: three tables

| Table | Required columns | Purpose |
|---|---|---|
| 利用実績 (billing detail) | Required: 利用者名 / 品目 / 数量 / 単価 / 金額<br>Optional: サービス日 | What was actually billed this month |
| 契約単価表 (contract prices) | Required: 利用者名 / 品目 / 契約単価<br>Optional: 適用開始日 | May be segmented by effective date (price revisions are routine) |
| 計画明細 (monthly plan) | Required: 利用者名 / 品目 | **Without this, missing-row and extra-row checks are disabled entirely** |

**What happens when a table is missing:**

- No **契約単価表** → every row is reported as "not in contract", which looks like noise
- No **計画明細** → missing-row and extra-row checks are disabled; 7 of 8 types remain
- No **利用実績** → nothing runs

Reconciliation means comparing two things against each other.
**With only one of them, there is nothing to reconcile.**

**Omitting the optional columns:** without `サービス日`, the date checks (outside period,
before contract start) are disabled. Without `適用開始日`, segmentation is unavailable and
multiple rows for the same item will raise an error.

**Do not translate the column names.** The code matches them literally as Japanese
strings: `利用者名` / `品目` / `数量` / `単価` / `金額` / `サービス日` / `契約単価` /
`適用開始日`. Replacing them with English headers will break loading.

### The eight discrepancy types

| # | Type | Location precision | Status |
|---|---|---|---|
| 1 | Unit price vs contract | row | ✅ segment selected by service date |
| 2 | Amount mismatch | row | ✅ reported alongside type 1, not merged |
| 3 | Missing row | row | ✅ judged against the plan table only |
| 4 | Extra row | row | ✅ always carries a specific reason |
| 5 | Consumption tax error | total | ⬜ **unavailable — tax-rate basis undecided** |
| 6 | Date outside period | row | ✅ |
| 7 | Not in contract | row | ✅ can be remembered by the rulebook |
| 8 | Unparseable date | row | ✅ separate section, excluded from the count |

"Service date precedes every contract segment" is **not** added as a ninth type. It is
folded into type 7 and distinguished by a textual sub-reason (`【契約未カバー…】`).
From the customer's point of view both mean "this money has no contractual basis",
and an extra type is one more concept they have to learn.

### The core mechanism: the rulebook

Every human confirmation **appends** a record; nothing is overwritten:

```
{customer, discrepancy type, content fingerprint, verdict, effective month, amount at confirmation}
```

When a similar discrepancy appears the following month, a three-tier threshold decides
whether the customer should be bothered again:

```
change > ¥10,000                        → return for human confirmation
otherwise change > 20% AND amount > ¥1,000 → return for human confirmation
anything else                           → no notification ("confirmed, no change")
```

**Why not a pure percentage rule?** 183,000 → 150,000 is an 18% drop, which a percentage
rule would let through — but it means the customer is short ¥33,000 while the report says
"nothing wrong this month." The three-tier rule catches it. Conversely 500 → 620 is a 24%
rise but only ¥120; financially that is not worth waking anyone up.

**The fingerprint must carry numeric context.** An earlier version keyed only on
`(customer, item)`, so once "Suzuki's 1,300 is an approved exception" was confirmed, a
genuine data-entry error of 1,500 the next month was silently waved through.
A fingerprint now looks like `unitprice|鈴木 次郎|訪問介護（生活援助2)|1,300|1,250`.

### Verdict states (three)

| State | Label | Meaning |
|---|---|---|
| `NEW` | needs confirmation | First sighting, or value differs from history |
| `CONFIRMED_SAME` | confirmed, no change | Same value, previous verdict retained |
| `RECURRING` | fixed before, recurred | The same error was fixed last time and is back |

A fourth state `CONFIRMED_DIVERGED` existed and was **deleted** — it never appeared once
across the full flow. A state that only exists in tests and never in real business, when
written into customer-facing material, is a promise the customer can never see happen.

### Known limitations (read this first)

**These are things this tool cannot do. Read them before deciding whether it fits:**

1. **It has never seen a single real Japanese billing document.** Normalization and every
   rule were validated against synthetic data. Real documents are messier: stamps
   obscuring text, handwritten amounts, tables spanning pages.
2. **Location precision stops at "row N", not "page N, row M, drawn as a box".**
   Coordinate-level pointing requires integrating actual PDF layouts.
3. **Tax validation is unavailable.** In long-term-care billing the user-borne portion and
   the insurance-covered portion carry different tax rates, and the per-row rate basis is
   undecided. Emitting a wrong tax figure is worse than emitting none.
4. **The amount for a missing row is derived from the contracted unit price**, so a contract
   price revision can be misread as "something changed" and trigger an extra confirmation —
   that is, one extra question per revision, not a missed discrepancy.
5. **Some cases in `final_check.py` were carried over from earlier self-test scripts** and
   their expected values have not been individually reviewed. When something breaks, suspect
   those first. B1–B4 in `review_gate.py` were written from scratch and individually checked.

### Running it

```bash
python final_check.py     # acceptance: 31 checks, exit code 0 means pass
python review_gate.py     # four blocking checks (B1–B4)
python make_demo.py       # generate the demo report into out/
```

Only dependency: `openpyxl`. **It runs fully offline** — `engine.py` and `rulebook.py`
contain no network calls, deliberately, so a customer can run it on their own machine.

**Acceptance criterion:** the number of `行目` occurrences must equal the number of data-row
statuses (both 6), and Chinese residue must be zero. Those two counts describe the same set
of discrepancies; if they diverge, the statistic has pulled in the legend or summary rows.

### Development log

The most valuable output of this project is not the tool itself but **a list of the
conditions under which it would produce a false conclusion.** All of the following were
found by the team itself, before any real customer or user existed:

- It reported "nothing wrong this month" while the customer was under-billing by ¥33,000
- It remembered one judgement and then waved everything through forever, even at 100× the amount
- It used `\u` escapes so the checker reported "no Chinese residue" while the report was still Chinese
- The report legend listed a verdict state that can never occur
- The acceptance script's own expected values were inverted

The log is kept for one reason: **knowing where it is unreliable matters more than knowing
where it is convenient.**
## 中文

### 这是什么

不读 PDF、不做 OCR、不碰客户的录入 —— 只回答一个问题：

> **这个月的账，错在哪里、错在第几行、上次确认过没有。**

输入三张表，输出一份 Excel 报告。第一行是「本月 N 处不一致，其中 M 处需要人工确认」，
下面逐条列出「第几行、谁、哪个服务项、单子写多少、合同应该是多少」。

### 为什么做这个

单据识别（OCR + 结构化）已经是便宜标配，freee、MoneyForward 都在做，而且越用越准。
但它们只负责**把单据变成数据**，不负责**按客户自己的业务规则核对**。

对账这一步，每家的规则都不一样：合同单价什么时候改的、哪一项这个月本来就不该出、
上一笔客户为什么手工放过了。这些事情 SaaS 大厂不会为单个客户去写，只能靠人。

本工具的定位就在这里：**不跟识别竞争，只做核对层。**
而且要能**记住客户上一次的判断** —— 这才是它比手工对账强的地方：人记不住上个月怎么判的。

### 实际输出

一份 Excel（`out/演示用_突合レポート_2026-10.xlsx`，附带 PNG 预览图）：

```
突合レポート　デモ用（模擬データ）　対象期間 2026-10
※模擬データです。実在の法人・利用者・会計データは一切含まれていません。
今月 5 件の不一致、うち 5 件は要確認（…）

#  差異種別              該当箇所                        該当行の値      推奨修正値 / 判定理由
1  単価と契約の不一致      6 行目 鈴木 次郎 / 訪問介護…     単価 1,300     契約単価 1,250（金額 6,250）
2  契約にない項目          9 行目 伊藤 五郎 / 緊急訪問介護…  単価 5,000     契約単価表に該当なし
3  余分な行               10 行目 渡辺 六郎 / 通院介助      数量 2 / 金額 4,000  今月の計画に含まれていません
4  契約開始より前の日付     11 行目 中村 七子 / 訪問介護…     2026-01-20   【契約未カバー】契約は 2026-04-01 から
5  金額の不一致           14 行目 木村 十郎 / 訪問介護…     金額 9,500    数量×単価 = 9,150（差 -350）
合計 5 件
```

### 输入：三张表

| 表 | 必需列 | 说明 |
|---|---|---|
| 利用実績（请求明细） | 必需：利用者名 / 品目 / 数量 / 単価 / 金額<br>可选：サービス日 | 本月实际出账的明细 |
| 契約単価表（合同单价） | 必需：利用者名 / 品目 / 契約単価<br>可选：適用開始日 | 可按生效日分段（改价是常态） |
| 計画明細（本月计划） | 必需：利用者名 / 品目 | **没有这张表，缺失行/多余行整组禁用** |

**缺哪张表会怎样：**

- 缺**契約単価表** → 每条都报「合同未约定」，客户会以为我们在瞎报
- 缺**計画明細** → 缺失行、多余行整组禁用，8 类只剩 7 类可用
- 缺**利用実績** → 跑不起来

对账的本质是拿两样东西相互比对。**只有一样，就没有对账。**

**省略可选列的后果：** 不写 `サービス日`，日期类检查（不在期间内、早于合同生效）失效；不写 `適用開始日`，分段不可用，同一品目出现多行会直接报错。

**列名请保持日文原样，不要翻译。** 代码是拿这些字符串做完全匹配的：`利用者名` / `品目` / `数量` / `単価` / `金額` / `サービス日` / `契約単価` / `適用開始日`。换成英文表头会直接加载失败。

### 八类差异

| # | 类型 | 定位精度 | 状态 |
|---|---|---|---|
| 1 | 单价与合同不一致 | 到行 | ✅ 分段按服务日取当期那段 |
| 2 | 金额不符 | 到行 | ✅ 与第 1 类并列输出，不合并 |
| 3 | 缺失行 | 到行 | ✅ 只按计划表判定 |
| 4 | 多余行 | 到行 | ✅ 必须带具体理由 |
| 5 | 税额计算错误 | 到合计 | ⬜ **不可用·税率口径待定** |
| 6 | 日期不在期间内 | 到行 | ✅ |
| 7 | 合同未约定 | 到行 | ✅ 可被规则簿记住 |
| 8 | 日期无法解析 | 到行 | ✅ 独立段落，不计入总数 |

「日期早于合同所有分段」不单开第 9 类，并入第 7 类，用文案子原因 `【契約未カバー…】` 区分。
客户视角下两者都是「这笔钱没有合同依据」，多一个类型客户就多学一个概念。

### 核心机制：规则簿

每次人工确认只**追加**一条记录，不覆盖：

```
{客户, 错误类型, 内容指纹, 判定, 生效月份, 确认时金额}
```

第二个月同类差异命中时，按三档阈值决定要不要再打扰客户：

```
变动 > 10,000 日元              → 退回，重新要人确认
否则 幅度 > 20% 且 金额 > 1,000 → 退回
其余                            → 不打扰（标「已确认过·建议不变」）
```

**为什么不是纯百分比：** 183,000 → 150,000 只跌 18%，纯百分比会放过，
但客户少收了 33,000 日元，而报告会写「本月没问题」。三档能挡住它。
反过来 500 → 620 涨 24% 只差 120 日元，三档不打扰 —— 财务上不值得为 120 日元叫醒客户。

**指纹必须带数值上下文。** 早先版本指纹只有 `(客户, 品目)`，
结果是「铃木单价 1,300 被认可为特例」之后，下个月真的录错成 1,500 也会被规则挡住。
现在的指纹形如 `unitprice|鈴木 次郎|訪問介護（生活援助2)|1,300|1,250`。

### 判定状态（三态）

| 状态 | 文案 | 含义 |
|---|---|---|
| `NEW` | 需确认 | 没见过，或数值与历史不同 |
| `CONFIRMED_SAME` | 已确认过·建议不变 | 同一数值，上次判定保留 |
| `RECURRING` | 上次已改，本次再现 | 上次改掉了，这次同样的错又出现 |

第四态 `CONFIRMED_DIVERGED` 曾经存在，后来**删掉了** —— 跑完整流程一次都没产出过。
一个只在测试里出现、业务中永远不出现的状态，写进对外材料就是给客户一个用不到的承诺。

### 已知边界（重要）

**这些是已知做不到的事，请务必先读这一段再判断它能不能用：**

1. **它没见过任何一份真实的日本请求书。** 归一化和各类判定都是在自造数据上验证的。
   真实票据只会更乱：印章遮字、手写金额、跨页表格。
2. **原文定位只到「第 N 行」，不是「第 N 页第 M 行划线框」。** 坐标定位需要接入真实 PDF 版式。
3. **税额校验不可用。** 介护报酬里利用者负担和保险给付税率不同，逐行税率口径未定，
   输出一个错误的税额比不输出伤害更大。
4. **缺失行的金额是由合同单价推算的**，合同调价会被误判成「有变动」而多问一次。
5. **`final_check.py` 里有一部分用例是从早期自测脚本搬过来的**，期望值没有逐条复核过。
   出问题时先怀疑这些，`review_gate.py` 的 B1–B4 是从头写并逐条核过的。

### 跑起来

```bash
python final_check.py     # 验收：31 项 + 阻断项，退出码 0 才算通过
python review_gate.py     # 四条阻断性检查（B1–B4）
python make_demo.py       # 生成演示报告到 out/
```

依赖：`openpyxl`。**全程断网可跑**（`engine.py` / `rulebook.py` 里没有任何网络调用），
这是刻意的：客户要能自己在本机跑。

**验收口径：** `行目` 出现次数必须等于 `数据行状态` 次数（同为 6），中文残留为 0。
这两个数描述同一批差异，对不上说明统计把图例或摘要混进来了。

### 开发过程记录

这个项目最大的产出不是工具本身，而是**一份「什么情况下它会给出假结论」的清单**。
以下缺陷全部在没有任何真实客户、没人使用的情况下自己翻出来的：

- 客户少收 33,000 日元时，报告显示「本月没问题」
- 记住一次判断后永久放行，金额涨 100 倍也不吭声
- 用 `\u` 转义让检查器显示「无中文残留」，而报告里还是中文
- 报告图例列着一个永远不会出现的判定状态
- 验收脚本自己的期望值填反

留着这份记录，是因为：**知道它哪里不可靠，比知道它哪里好用更重要。**

---


---

*模擬データです。実在の法人・利用者・会計データは一切含まれていません。*
*Synthetic data only. No real organization, individual, or accounting record is included.*
*模拟数据，不含任何真实法人、利用者或账目信息。*