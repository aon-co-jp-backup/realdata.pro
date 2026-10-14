# realdata.pro

**誰でも使える、Rust製・オープンソースのデータ分析プラットフォーム**(SAS Viyaの考え方を参考にした自由実装)。
大企業にも、中小企業にも、これから起業する人にも。CSVをドラッグするだけで、クレンジング・集計・予測まで。

公式サイト(準備中): https://realdata.pro

> SAS / SAS Viya は SAS Institute Inc. の商標です。本プロジェクトは SAS Institute とは無関係で、
> 同社のコード・API・仕様を複製するものではありません。機能の方向性のみを参考にしています。

## 🌍 世界リサーチ

テーマ(例:「抹茶の海外需要」「電気自動車の充電インフラ」)と国・地域を選ぶだけで、次のことを自動で行います。

1. 各国の現地語で Web とニュースを集める
2. テーマと無関係な情報を除く
3. AI(aruaru-llm の無料 AI)が、トレンド・機会・リスク・具体的な提案・国別の論調をまとめる

根拠はすべて、実際に集めた記事へのリンクで示します。レポートは日本語・英語と約130言語から選べます。
集めた記事はデータセットになり、集計やグラフにもそのまま使えます。

> **毎朝の自動収集の試験(2026-09-26)**: 日本全国48か所を対象に試験中。1か所目(岩手県)は27件の検索で
> 70件を収集(以前は0件で失敗)。既定は1日600件(約22か所/日)で、全国一巡に約2日の見込み。詳細は
> [PORTING.md](PORTING.md)・[CLAUDE.md](CLAUDE.md)を参照。
>
> *Automatic daily collection test (2026-09-26, English): testing across 48 locations in Japan. The first
> location (Iwate) collected 70 records from 27 searches (previously failed with 0). Default is 600
> searches/day (~22 locations/day), an estimated ~2 days for a full national pass. See
> [PORTING.md](PORTING.md) / [CLAUDE.md](CLAUDE.md) for details.*

## aon-co-jp エコシステムで構成

| 役割 | 使うもの | 状態 |
|---|---|---|
| 分析エンジン(列指向インメモリ DataFrame) | `rrd-core`(本リポジトリ、外部依存なし) | ✅ |
| 行列演算(重回帰など) | [open-cuda](https://github.com/aon-co-jp/open-cuda) `opencuda-blas` の GEMM | ✅ CPU バックエンドで実働 |
| Web サーバー・API | [RPoem](https://github.com/aon-co-jp/RPoem) `open-runo-poem-compat` + GraphQL 単一エンドポイント | ✅ |
| ネットワーク基盤 | [open-web-server](https://github.com/aon-co-jp/open-web-server)(RPoem 経由で組込み)、本番の前段ゲートウェイ | ✅ 組込み / 🔜 realdata.pro 前段 |
| データの永続化・版管理 | [aruaru-db](https://github.com/aon-co-jp/aruaru-db)(Git-on-SQL、`AS OF COMMIT` で過去の分析を再現) | 🔜 次段 |
| 検索ワードからの取り込み(Google / YouTube / GitHub) | [aruaru-llm](https://github.com/aon-co-jp/aruaru-llm) `POST /v1/search/raw` | ✅ |
| 調査対象 URL からの取り込み(CSV・JSON・HTML の表) | 本リポジトリ(SSRF 対策付き) | ✅ |
| AI による分析の説明(日本語・英語と約130言語から選択) | [aruaru-llm](https://github.com/aon-co-jp/aruaru-llm) | ✅ |
| グラフ描画 | 棒・円グラフ(ブラウザ SVG)と、[open-directx](https://github.com/aon-co-jp/open-directx)(Vulkan)による GPU 描画。GPU が無ければ AVX-512 / AVX2 の CPU 描画 | ✅ |
| データ保管庫(GitHub 非公開リポジトリ)の容量管理 | 1リポジトリ10GB上限に備え、索引リポジトリ設定時は容量が近づくと新しいリポジトリへ自動で引っ越す | ✅ 2026-09-28 |

## SAS Viya の特徴との対応

| SAS Viyaの特徴 | realdata.pro | 状態 |
|---|---|---|
| インメモリ分散処理エンジン(CAS) | 列指向 DataFrame + open-cuda | ✅ 単一ノード |
| データ準備・クレンジング | 欠損補完・欠損行/重複行除去・フィルタ・並べ替え・列選択 | ✅ |
| 探索的データ分析 | 要約統計(四分位など)・相関 | ✅ |
| 集計 | group by(count/sum/mean/min/max/median/std) | ✅ |
| 予測・機械学習 | 重回帰(open-cuda GEMM) | ✅ 最初の一歩 |
| ノーコード GUI | Web UI(ファイル選択だけで使える) | ✅ |
| Python / R 連携 | GraphQL で任意の言語から / 専用バインディング | ✅ GraphQL / 🔜 専用 |

## すぐ試す

```bash
# 1. エコシステムの依存をコミット固定で .deps/ に取得(作業ツリーには触れない)
bash scripts/fetch-deps.sh          # Windows: powershell -File scripts\fetch-deps.ps1
# 2. サーバー起動 → ブラウザで http://127.0.0.1:4701/
cargo run --release -p rrd-server
```

コマンドラインだけで使う場合:

```bash
cargo run --release -p rrd-cli -- describe examples/sales.csv
cargo run --release -p rrd-cli -- groupby examples/sales.csv city sales:sum qty:mean
```

GraphQL(`POST /graphql`)の例:

```graphql
mutation { loadCsv(name: "sales", csv: "city,sales\nTokyo,100\nOsaka,80\n") { rows } }
query { regression(name: "sales", target: "sales", features: ["qty", "ad_cost"]) { intercept coefficients rSquared device } }
```

## 保管庫のお引越し(自動)

収集データは GitHub の非公開リポジトリ(保管庫)に保存します。1リポジトリが約1GBに近づく(増加ペースから14日以内と予測した)と、
新しい非公開リポジトリ(`realdata-archive-2` …)を自動で作り、書き込み先を切り替えます。古いデータは移動せず、読み取り専用のまま残ります。
索引リポジトリ `RRD_CATALOG_REPO` を設定すると有効になります(未設定なら単一リポジトリ)。詳細は [PORTING.md](PORTING.md)。

## 構成

```
crates/
  rrd-core/     分析エンジン(外部依存なし)
  rrd-compute/  open-cuda による行列演算
  rrd-server/   RPoem 上の GraphQL + Web UI
  rrd-cli/      コマンドライン `rrd`
deps.lock       依存するエコシステムのリポジトリとコミット
scripts/        fetch-deps(.deps/ への固定取得)
```

開発では `cargo fmt-own` / `cargo lint-own` / `cargo test-own` を使います(`cargo fmt --all` は依存先まで整形するため使いません)。

## English

**realdata.pro** is an open-source analytics platform in Rust, inspired by SAS Viya (not affiliated with
SAS Institute), built on the aon-co-jp ecosystem: a dependency-free columnar engine, open-cuda GEMM for
regression, and a RPoem-based server exposing a single GraphQL endpoint plus a no-code web UI. aruaru-db
persistence, aruaru-llm explanations and open-directx rendering are next. See [PORTING.md](PORTING.md).

**Automatic archive rotation ("hikkoshi").** Collected data is stored in private GitHub repositories. When the current
shard nears ~1GB (or is predicted to within 14 days), a new private repository (`realdata-archive-2`, …) is created
automatically and writes switch to it; old shards stay read-only. Enabled by setting the catalog repository
`RRD_CATALOG_REPO`; otherwise a single repository is used. Details in [PORTING.md](PORTING.md).

## ライセンス

MIT OR Apache-2.0([LICENSE-MIT](LICENSE-MIT) / [LICENSE-APACHE](LICENSE-APACHE))

## JSON処理の使用方針: `parse_secure_sonic`(速い型+安全)と`sonic-rs`は使える所では全て使う(2026-10-14、ユーザー指示)
正本は[`RS-JSON`](https://github.com/aon-co-jp/RS-JSON)(`README.md`・`PORTING.md`)。

**方針**: JSONを読む所は、使える所では**全て** `rust-json`(RS-JSON)の `parse_secure_sonic`(`fast`機能、結果は`sonic_rs::Value`)を使う。
`sonic-rs` を直接使える所(型付き構造体への読み込みなど)も同様。使えない所は `parse_secure`(結果は`serde_json::Value`)を使う。
外部から来る信頼できない入力(HTTP本文・ネットワーク越しのファイル・アップロード)は、**必ずサイズ・深さ・重複キーの上限つき**で読む。

**実測(7 MB・4万件)**: `serde_json` 200 ms / `parse_secure` 217 ms / **`parse_secure_sonic` 83 ms(約2.4倍速い、防御込み)** / `sonic-rs`そのまま 33 ms(約6倍、防御なし)。
`serde_json::Value`に読む限りSIMDでもほぼ速くならない(188 ms)。**速さの源は、結果を連続した領域に詰め込む値の型**。

**使える所 / 使えない所**
- ✅ 読み取り中心で、JSONを大きく・頻繁に読む所(キャッシュJSON、設定、APIの応答、ログ、インデックス)。
- ✅ 外部からの入力の入口(速さより防御が目的。`parse_secure`でも可)。
- ⚠️ `serde_json::Value`を関数間で受け渡している既存コードは、型を`sonic_rs::Value`に直す書き換えが要る(`JsonValueTrait`/`JsonContainerTrait`のメソッド)。書き換えずに`serde_json::Value`へ読むと速くならない。
- ❌ WASM(`rust_json::light`、依存ゼロ)、PHP/JavaScript側のJSON処理(Rustの部品は使えない)。
- 注意: 依存先の`sonic-rs`は内部で`unsafe`を使う(自前コードに`unsafe`は無い)。小さなJSON(数KB)では体感差が出ない。速度目的の書き換えは、実測で効く所から。

**導入**: `Cargo.toml`に `rust-json = { path = "../RS-JSON", features = ["fast"] }`。使用例は`RS-JSON/README.md`。

**このリポジトリでの適用状況**: 未適用(方針のみ)。コードの切り替えは、JSONを大きく・頻繁に読む所から実測して順に行う。

## JSON handling policy: use `parse_secure_sonic` (fast type + secure) and `sonic-rs` everywhere they can be used (2026-10-14, user instruction)
The source of truth is [`RS-JSON`](https://github.com/aon-co-jp/RS-JSON) (`README.md`, `PORTING.md`).

**Policy**: wherever JSON is read, use `parse_secure_sonic` from `rust-json` (RS-JSON, the `fast` feature; the result is `sonic_rs::Value`) **everywhere it can be used**.
Use `sonic-rs` directly where possible too (e.g. reading into typed structs). Where it cannot be used, use `parse_secure` (result: `serde_json::Value`).
Untrusted input from outside (HTTP bodies, files over the network, uploads) **must always be read with size / depth / duplicate-key limits**.

**Measured (7 MB, 40,000 records)**: `serde_json` 200 ms / `parse_secure` 217 ms / **`parse_secure_sonic` 83 ms (about 2.4x faster, with the defences)** / raw `sonic-rs` 33 ms (about 6x, no defences).
Reading into `serde_json::Value` gives almost no speed-up even with SIMD (188 ms). **The speed comes from the value type that packs the result into contiguous memory.**

**Where it can / cannot be used**
- ✅ Read-heavy code that parses large or frequent JSON (cache JSON, config, API responses, logs, indexes).
- ✅ Entrances for outside input (the goal there is defence rather than speed; `parse_secure` is fine).
- ⚠️ Existing code that passes `serde_json::Value` between functions needs its type changed to `sonic_rs::Value` (methods from `JsonValueTrait` / `JsonContainerTrait`). Reading into `serde_json::Value` without that change is not faster.
- ❌ WASM (`rust_json::light`, zero dependencies) and the JSON handling on the PHP / JavaScript side (Rust components cannot be used there).
- Note: the `sonic-rs` dependency uses `unsafe` internally (our own code has none). For small JSON (a few KB) the difference is not noticeable; do speed-motivated rewrites where measurements show a benefit.

**Adoption**: in `Cargo.toml`, `rust-json = { path = "../RS-JSON", features = ["fast"] }`. Example in `RS-JSON/README.md`.

**Status in this repository**: Not yet applied (policy only). Code will be switched over place by place, starting where measurements show large or frequent JSON reads.
