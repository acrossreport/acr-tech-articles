# C# / Node.js から Docker 上の Linux で動く爆速帳票 PDF 生成環境を作る

## やりたいこと

帳票 PDF 生成を担うコンポーネントだけを Linux コンテナに切り出し、C# や Node.js のアプリケーションから呼び出せるようにしたい——というケースは増えています。Windows サーバーのライセンス費・運用コストを下げたい、Kubernetes 上でオートスケールさせたい、といった動機が典型です。

本記事では、筆者が開発している Rust 製帳票エンジン ACR（Across Report）を例に、「ネイティブレンダラーを Docker + Linux 環境で動かし、C# / Node.js の既存コードからほぼそのまま呼び出す」構成を紹介します。ポイントは、**ブラウザも .NET ランタイムもコンテナに含めず、ネイティブバイナリ一本で完結させる**ことです。

## 全体構成

```
┌─────────────────────────┐
│ C# アプリ / Node.js アプリ │
└───────────┬─────────────┘
            │ FFI (axr_layout.so)  /  N-API (acr_node_native.so)
┌───────────▼─────────────┐
│  Docker コンテナ (Linux)  │
│  ├ acr_core              │ ← JSON テンプレート → RenderCommand
│  ├ acr_renderer_skia     │ ← RenderCommand → PDF/PNG (Skia)
│  └ fonts-vlgothic        │ ← 日本語フォント
└─────────────────────────┘
```

呼び出し側言語によって、コンテナに積むものが少し変わります。

- **C# から呼ぶ場合**: `libaxr_layout.so` を P/Invoke で呼び出す（Windows の `axr_layout.dll` と同じ FFI インターフェース）
- **Node.js から呼ぶ場合**: `acr_node_native`（napi-rs 製のネイティブアドオン）を `require` して直接呼び出す

いずれの経路でも、PDF/PNG の実体を生成しているのは同じ Rust バイナリです。

## ビルドの勘所

### 1. Rust 本体のビルド

```bash
cargo build --release
```

これで `acr_core` / `acr_renderer_skia` を含む全クレートが Linux 向けにビルドされます。Windows ARM64 のような特殊な対応は Linux x86_64 では不要で、`skia-safe` のプリビルドバイナリがそのまま使えます。

### 2. Node.js から使う場合（napi-rs）

Node.js 向けのネイティブアドオンは `acr_node_native` クレートが担っています。これは N-API（`napi` / `napi-derive`）を使った `cdylib` で、JS 側からは次のような関数が直接呼べます。

```ts
// index.d.ts（抜粋）
export function renderPdf(templateJson: string, dataJson: string, outputPath: string): number
export function renderPreview(templateJson: string, dataJson: string): Array<PageImage>
```

ビルドは次の手順です。

```bash
cd acr_node_native
npm install
npx napi build --platform --release
```

ワークスペース側で `target-dir` をカスタマイズしている構成だと、`napi build` の後処理が `.node` ファイルを見つけられず失敗することがあります。その場合は

```bash
cargo build --release -p acr_node_native
# 生成された acr_node_native.so を index.<platform-triple>.node / index.node にコピー
```

と手動でリネーム・配置すれば回避できます。Docker のマルチステージビルドではこの手動コピー手順をそのまま `Dockerfile` に落とし込むとハマりにくいです。

### 3. Dockerfile の例（Node.js から利用する場合）

```dockerfile
# ---- build stage ----
FROM rust:1.78-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release -p acr_node_native

# ---- runtime stage ----
FROM node:20-bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
        fonts-vlgothic \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/libacr_node_native.so ./index.node
COPY acr_node_native/index.js acr_node_native/index.d.ts ./
COPY acr_node_native/package.json ./

# アプリ本体をここに COPY し、require('./index.node') 経由で呼び出す
```

ポイントは2つです。

- **ビルドステージとランタイムステージを分離する**ことで、最終イメージに Rust ツールチェーンを残さず軽量化する
- **日本語フォントはランタイムステージにだけ** `apt install fonts-vlgothic` で入れる。ビルドステージには不要

### 4. C# から呼ぶ場合

C# アプリ自体が .NET (Core) でクロスプラットフォーム対応済みであれば、`libaxr_layout.so` を P/Invoke で参照するだけで Linux 上でも同じ呼び出しコードが動きます。

```c
char* render_html(const char* template_json, const char* data_json);
void free_string(char* ptr);
```

.NET Framework 専用アプリの場合は、まず .NET (Core) / .NET 8 以降への移行が前提になりますが、帳票レンダリング部分の呼び出しインターフェース自体は Windows 版と同一のため、「帳票エンジンだけ先に Linux 化して検証し、アプリ本体の .NET 移行は後から進める」という段取りも可能です。

## なぜこの構成だと「軽い」のか

Chrome ベースの PDF 生成（Puppeteer 等）と比較した場合、この構成の強みは次の点です。

- ランタイムイメージに Chromium 一式（数百MB）を含めずに済む
- レンダラープロセスという概念がないため、ページ数に応じたプロセス管理・メモリ監視の複雑さが発生しない
- Rust バイナリ単体で完結するため、コンテナの起動も速く、オートスケール時のコールドスタートが短い

## まとめ

- 帳票 PDF 生成だけを Linux コンテナに切り出し、C# / Node.js からネイティブ呼び出しする構成は、マルチステージビルド + 日本語フォントのランタイム同梱で実現できる
- Node.js からは napi-rs 製アドオン、C# からは FFI (`.so` / `.dll` 共通インターフェース) という2つの経路が用意されていれば、呼び出し側の言語を問わず同じエンジンを使い回せる
- ブラウザエンジンに依存しないことで、イメージサイズ・起動速度の両面で有利になる

---

【お知らせ・個別相談】
本記事で紹介した ACR（Across Report）を活用した**既存帳票システムのマイグレーション（ActiveReports等からの移行）**や、Puppeteer 構成からのサーバーコスト削減コンサルティングを承っています。
「高額なライセンス更新費に悩んでいる」「Linux / クラウド環境へ安く移植したい」といった課題がございましたら、[Across Systems お問い合わせ](https://acrossreport.com) よりお気軽にご相談ください。
