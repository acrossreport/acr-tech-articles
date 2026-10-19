# ActiveReports の既存資産を活かして Web 移行・Linux 化するアプローチ

## 背景: ActiveReports 資産が抱える3つの悩み

.NET Framework 時代の業務システムで帳票出力に ActiveReports（GrapeCity / MESCIUS）を使っているプロジェクトは今も多くあります。長年運用されてきた分、現場には次のような悩みが溜まりがちです。

1. **ライセンス更新費が年々重くのしかかる** — サーバーライセンスやランタイムライセンスの更新費用が、帳票機能単体の価値に見合わなくなってきている
2. **Windows / .NET Framework に縛られている** — ActiveReports 自体や、それをホストする Web アプリが .NET Framework 前提で、Linux コンテナやクラウドネイティブな構成に乗せ替えられない
3. **デザイナー資産（帳票レイアウト）が何十〜何百種類もある** — 今さら全部作り直すのは現実的ではなく、「資産を活かしたまま」が絶対条件になる

この3つが同時に来ると、「全面リプレイス」も「現状維持」もどちらも選びにくく、プロジェクトが膠着しがちです。

## 移行のアプローチ: 「置き換え」ではなく「意味的互換」を狙う

ActiveReports のレイアウトファイル（.rdlx など）をそのままバイナリ互換で読み込む、というアプローチは現実的ではありません。フォーマットの再現性を完全に保証するコストが高すぎるからです。

そこで筆者が開発している ACR（Across Report）では、**ActiveReports の「概念・振る舞い」を JSON テンプレートのレベルで再現する**という方針を取っています。具体的には以下のような互換性を実装レベルで持たせています。

### セクション構造の互換

ActiveReports の `Detail` / `PageHeader` / `PageFooter` などのセクション概念をそのまま継承し、セクションごとの `NewPage AFTER`（セクション出力後に改ページする）といった制御も同じ意味で再現しています。

```rust
// acr_core/src/layout/engine.rs
// ② ActiveReports互換：NewPage AFTER
if sec.new_page_after {
    ...
}
```

### 日付書式・フォント指定の互換

`.NET` 由来の `yyyy/MM/dd` `HH:mm:ss` といった書式記号をそのまま解釈できるようにしているため、既存テンプレートの書式文字列を事実上そのまま移植できます。フォント指定も、ActiveReports の `Font` / `FontName` フィールドと CSS 的な `Style` の `font-family` の優先順位を、ActiveReports の挙動に合わせて解決しています。

### バーコード種別の互換

ActiveReports 独自のバーコード種別コード（例: 旧コード `18`）も、現行の `Code128B` 等へマッピングする形で吸収しており、データ側の変更を最小限に抑えられます。

これらはいずれも「1ドットも違わない再現」が目的ではなく、**既存の帳票定義・データ・運用知識を、作り直しなしでそのまま移せる**ことを目的にした互換性です。

## なぜ Linux 化できるのか

ACR は Rust + Google Skia で実装されたネイティブレンダラーで、.NET Framework にも Windows API にも依存しません。

- レンダリングエンジン本体（`acr_core` / `acr_renderer_skia`）は Windows / macOS / Linux で同一のバイナリ構成でビルド可能
- Linux では日本語フォント（VL Gothic 等）を `apt install fonts-vlgothic` で用意するだけで、Windows と同一の出力が得られる
- 既存の .NET / C# 資産からは FFI 経由（`axr_layout.dll` / `libaxr_layout.so`）で呼び出せるため、呼び出し側のコードをいきなり全部書き換える必要がない

```c
// FFI インターフェース（axr_layout.dll / libaxr_layout.so 共通）
char* render_html(const char* template_json, const char* data_json);
void free_string(char* ptr);
```

つまり移行は「C# アプリを捨てる」のではなく、「帳票レンダリング部分だけを Rust/Skia 製の軽量エンジンに差し替え、呼び出し側は FFI 経由でそのまま使う」という段階的な置き換えが可能です。Windows でも Linux でも同じ FFI インターフェースで動くため、最終的に Web アプリ本体を Linux コンテナへ移す際も、帳票部分だけ先行して検証・切り替えができます。

## 典型的な移行ステップ

1. **棚卸し**: 既存 ActiveReports テンプレートの一覧化。セクション構成・バーコード種別・特殊書式（割賦・ルビ・縦書き等）の有無を確認
2. **JSON テンプレート化**: 優先度の高い帳票から、ActiveReports の定義を ACR の JSON テンプレートへ変換（セクション・フィールド単位の機械的な変換が中心になるケースが多い）
3. **並行稼働での検証**: 既存の ActiveReports 出力と ACR 出力を同一データで突き合わせ、WYSIWYG プレビューで差分を目視・自動比較
4. **呼び出し側の切り替え**: C# 側は FFI 呼び出しに差し替えるだけで済むため、Web フロント・業務ロジックには手を入れない
5. **Linux 化**: 検証が済んだ帳票から順に、レンダリング部分を Linux コンテナへ移設

## まとめ

- ActiveReports 資産の移行は「全部作り直す」か「何もしない」かの二択ではない
- セクション構造・日付書式・バーコード種別など、ActiveReports の「振る舞い」を意味的に継承したエンジンに乗せ替えることで、テンプレート資産を活かしたまま移行できる
- レンダリング部分だけを FFI 経由で差し替えられるため、Windows → Linux 化も段階的に進められる

---

【お知らせ・個別相談】
本記事で紹介した ACR（Across Report）を活用した**既存帳票システムのマイグレーション（ActiveReports等からの移行）**や、Puppeteer 構成からのサーバーコスト削減コンサルティングを承っています。
「高額なライセンス更新費に悩んでいる」「Linux / クラウド環境へ安く移植したい」といった課題がございましたら、[Across Systems お問い合わせ](https://acrossreport.com) よりお気軽にご相談ください。
