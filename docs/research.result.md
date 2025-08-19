# resources
- [research 2](https://chatgpt.com/share/e/68a32833-0b08-8006-8369-80dfa475642c)
- [research 1](https://chatgpt.com/share/e/68a32819-37c0-8006-b9eb-a79d9de07a9a)
- [mozc](https://github.com/copilot/c/8c6a3a23-8d80-4569-b785-0b0eefdd4b37)
---

## 結論（TL;DR）

* **推奨スタック（クロスプラットフォーム）**
  **Rustコア** + **各OS純正IME API**（Windows: **TSF**、macOS: **InputMethodKit**、Linux: **IBus/Fcitx5**）で入力エンジンを実装。
  ユーザー向けの「uTorrent風ローカルポータル」は**Tauri**（Rust/軽量WebView）で提供。
  機能拡張はモジュール式：

    * **リアルタイムタイプミス修正**：自前ルール + **LanguageTool**（ローカルHTTPサーバ）
    * **音声→テキスト**：**whisper.cpp**（完全ローカル）
    * **リアルタイム翻訳/テンプレ生成**：ローカルLLM（**llama.cpp**）＋任意クラウドのハイブリッド
    * **Trump card（“Jenny/Jenni AI”参照）**：文脈に応じた\*\*「Anchor Cards」\*\*（挨拶/結び/ビジネス文テンプレ）をIMEサジェスト行にカードUIで提示
      → **高速・安全（プロセス内/ローカル優先）・拡張容易**
      。WindowsはTSF要件・署名要件に準拠。([Microsoft Learn][1], [Apple Developer][2], [ibus.github.io][3], [GitHub][4], [dev.languagetool.org][5])

---

## 1) 候補ソリューション比較表

> ★GitHubスター数は**2025-08-18時点**で各リポジトリの表示値を記載

| Solution name                                                   | Programming language                          | Pros                                                                                                                    | Cons                                                     | Recommendation (5 pts) | GitHub star (if applicable)                                                                                       | Note                                                                                |
|-----------------------------------------------------------------|-----------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|-----------------------:|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| **Rust Core + OS-Native IME (TSF/IMK/IBus) + Tauri Portal**（推奨） | Rust（コア/IPC）、C++（TSFラッパ）、Swift/Obj‑C（IMKブリッジ） | 各OS公式APIで**互換性/信頼性/レイテンシ最小**。Portalは軽量な**Tauri**。音声**whisper.cpp**や**LanguageTool**、**llama.cpp**を**完全ローカル構成**でプライバシー強。 | OSごとのブリッジ開発コスト。TSF/コード署名などWindows配布要件。                   |                **5.0** | Tauri **95.6k**★、whisper.cpp **42.4k**★、llama.cpp **85.1k**★、LanguageTool **13.4k**★、IBus **942**★、Fcitx5 **2k**★ | Windows: **TSF**必須/署名必須。macOS: **IMK**バンドル。Linux: **IBus/Fcitx5**エンジン。([GitHub][6]) |
| **Rimeベース（librimeをフォーク）+ 既存フロントエンド**（Squirrel/ibus-rime等）       | C++（librime）、Swift（Squirrel）、他                | 豊富な辞書/スキーマ。中日等の形態素変換や候補提示が成熟。                                                                                           | コア設計がIME全体要求に最適化されているとは限らず、AI拡張の統合が重め。Windows向けは別途実装が必要。 |                **3.8** | librime **3.8k**★、Squirrel **5.1k**★、ibus‑rime **805**★                                                           | 既存資産の活用に向くが、Anchor独自体験の作り込み余地は制約あり。([GitHub][7])                                    |
| **Electron Portal + OS-Native IME**                             | TypeScript/JS（Electron）、Rust/C++/Swift（各ブリッジ） | Web人的資源を活用しやすい。周辺エコシステム豊富。                                                                                              | ランタイム重量（メモリ/起動）と配布サイズ。セキュリティ/自動更新設計の手間。                  |                **3.9** | Electron **118k**★                                                                                                | PortalのみElectronにしても良いが、**Tauriの方が軽量**。([GitHub][8])                                |
| **Windows先行MVP（TSFのみ）+ 最小Portal**                               | C++/Rust（TSF）、Tauri                           | 市場規模の大きいWindowsで早期β。TSF準拠で商用配布しやすい。                                                                                     | macOS/Linux遅延。TSF学習コスト。                                  |                **4.2** | （OS APIはN/A）Portal: Tauri **95.6k**★                                                                              | 署名/TSF要件への対応は必須。([Microsoft Learn][9], [GitHub][6])                                 |
| **macOS先行MVP（InputMethodKitのみ）+ 最小Portal**                      | Swift/Obj‑C（IMK）、Tauri                        | IMKは**実装が比較的素直**。UI品質を担保しやすい。                                                                                           | Windows/Linux遅延。IMKドキュメント/サンプルがやや古め。                     |                **4.1** | （OS APIはN/A）Portal: Tauri **95.6k**★                                                                              | Apple公式IMKドキュメント/サンプルあり。([Apple Developer][2], [GitHub][10])                        |
| **アクセシビリティAPIを用いた疑似キーボード/オーバーレイ**（非推奨）                          | TS/JS（Electron/Tauri）                         | IME APIに触れず実装可能。                                                                                                        | **真のIMEではなく**互換性/安定性/セキュリティ課題。多くのアプリで制限。                 |                **1.5** | Electron **118k**★                                                                                                | 研究用のみ推奨。商用品質の入力体験を満たしづらい。([GitHub][8])                                              |

> **出典（スター数/公式API）**
>
：Tauri（95.6k）・Electron（118k）・whisper.cpp（42.4k）・llama.cpp（85.1k）・LanguageTool（13.4k）・IBus（942）・Fcitx5（2k）・librime（3.8k）・Squirrel（5.1k）・ibus‑rime（805）。各GitHubページの現在値を参照。([GitHub][6])

---

## 2) 推奨アーキテクチャの要点

* **IMEコア**：Rustで共通ロジック（辞書・候補生成・変換・モジュールパイプライン）。

    * **Windows**：TSF（Text Services Framework）で実装。IMEの要件（**TSF対応・署名必須**）を満たす。
      `ITfThreadMgrEx::GetActiveFlags`でUWP/デスクトップ差を識別。([Microsoft Learn][9])
    * **macOS**：**InputMethodKit**の`IMKServer`等を利用したIMKバンドル。([Apple Developer][2])
    * **Linux**：**IBus**エンジン（将来的にFcitx5対応追加）。([ibus.github.io][3])
* **ローカルポータル（uTorrent風）**：**Tauri**（軽量/安全、各OSのシステムWebView活用）。([GitHub][6])
* **モジュール**（プロセス分離/IPCで連結）

    * **タイプミス修正/文法**：**LanguageTool**
      ローカルHTTPサーバ（Javaランタイム同梱/設定可能）。([dev.languagetool.org][5])
    * **音声入力**：**whisper.cpp**のC/C++バインディング。ローカルでストリーミング認識。([GitHub][11])
    * **翻訳/テンプレ生成/要約**：ローカル**llama.cpp**
      （必要に応じてクラウド回帰）。NLLB/Marian系で低遅延翻訳の選択肢。([GitHub][12], [Marian][13])
* **Trump card（“Jenny/Jenni AI”参照）**：**Anchor Cards**

    * 現在の入力文脈（件名/宛名/企業名/言語）から**挨拶・結び・承諾/依頼テンプレ**を候補列の先頭に**カードUI**
      で提示（キーボードのみで選択/展開）。アイデア元として**Jenni AI**
      の「文中オートコンプリート」体験を参考。([Jenni AI][14], [Jenni Help Center][15])

**アーキテクチャ（概念図）**

```
[Key Events] -> [OS IME API (TSF/IMK/IBus)] -> [Rust Core Pipeline]
  -> [Speller + Grammar (LanguageTool)]
  -> [Translator (NLLB/Marian or LLM)]
  -> [LLM (llama.cpp) for templates]
  -> [Candidate List + Anchor Cards] -> [Commit to App]
                                   ↘︎ [Portal (Tauri) settings/analytics]
```

**Pluginインターフェース例（英語のコード例）**

```rust
// Core plugin interface (Rust)
pub struct Context {
    pub locale: String,
    pub app_name: String,
    pub surrounding_text: String,
}

pub enum Action {
    ReplacePreedit(String),
    Commit(String),
    OfferCandidates(Vec<String>),
    Noop,
}

pub trait AnchorModule {
    fn name(&self) -> &'static str;
    fn transform(&mut self, ctx: &Context, input: &str) -> Action;
}

// Examples: LanguageToolModule, WhisperDictationModule, TranslateModule, AnchorCardsModule
```

---

## 3) 3人チーム（CEO/ジュニア開発1/パートタイム1）での詳細タイムライン

> 前提：**macOS先行MVP（8週間）→ Windows対応（+6週間）→ Linux対応（+3週間）**
> 稼働：CEO=プロダクト/仕様/QA/調達、ジュニア=実装中心（1.0FTE）、パートタイム=ビルド/CI/辞書/Tooling（0.5FTE想定）

### フェーズA：MVP（macOS + ポータル + 3機能）— **週1〜8**

| 週 | 目標/成果物             | 詳細タスク                                           | 担当                                 |
|---|--------------------|-------------------------------------------------|------------------------------------|
| 1 | 要件凍結・設計            | 仕様書/UXワイヤ（Anchor Cards含む）、IMK/Portal設計、データ/権限方針 | CEO主導、Dev支援                        |
| 2 | 骨格リポジトリ            | Rustコア雛形、IMKバンドル雛形、Tauri雛形、CI/CD                | Dev、PT                             |
| 3 | 変換/候補基盤            | プレエディット/候補表示、YAML辞書読み込み                         | Dev                                |
| 4 | **LanguageTool**統合 | ローカルHTTPサーバ起動/IPC、辞書ルール追加                       | Dev、PT ([dev.languagetool.org][5]) |
| 5 | **whisper.cpp**音声  | ホットキー録音→逐次確定、誤起動防止                              | Dev ([GitHub][11])                 |
| 6 | **Anchor Cards**   | 挨拶/結び/返信テンプレ（英/日）カード提示ロジック                      | CEO（文例）、Dev                        |
| 7 | Portal完成           | 設定（辞書/ショートカット/翻訳ON/OFF）、ログ/診断                   | PT、Dev ([GitHub][6])               |
| 8 | β配布                | 署名/インストーラ、クラッシュ収集、QA、ユーザテスト                     | CEO主導                              |

### フェーズB：Windows対応（TSF）— **週9〜14**

| 週  | 目標/成果物            | 詳細タスク                                | 担当                            |
|----|-------------------|--------------------------------------|-------------------------------|
| 9  | TSFブリッジ骨格         | `ITfThreadMgr`活性化/コンテキスト、プレエディット     | Dev ([Microsoft Learn][16])   |
| 10 | 候補UI/切替           | `GetActiveFlags`でUWP/デスクトップ分岐、既存コア接続 | Dev ([Microsoft Learn][17])   |
| 11 | 署名/配布準備           | コードサイニング証明書、インストーラ                   | CEO、PT ([Microsoft Learn][9]) |
| 12 | LanguageTool/音声連携 | macOSで実装済みモジュールを移植                   | Dev                           |
| 13 | QA/最適化            | レイテンシ/IME切替/ショートカット確認                | 全員                            |
| 14 | β配布               | 社内/限定ユーザ展開                           | CEO                           |

### フェーズC：Linux対応（IBus）— **週15〜17**

| 週  | 目標/成果物 | 詳細タスク                     | 担当                         |
|----|--------|---------------------------|----------------------------|
| 15 | IBus骨格 | `IBusEngine`実装、設定UI連携     | Dev ([ibus.github.io][18]) |
| 16 | 検証     | Wayland/X11、主要アプリでの動作確認   | Dev、PT                     |
| 17 | β配布    | パッケージ（.deb/.rpm/AppImage） | PT                         |

**リスク/緩和**

* **TSFの学習コスト**→ 先にIMKでMVPを完成させ、TSFは**最小要件**から段階適用。([Microsoft Learn][1])
* **署名/配布**→ WindowsのIME要件（**TSF対応・署名必須**）を早期に満たす計画と調達。([Microsoft Learn][9])
* **音声/翻訳の遅延**→
  まず\*\*ローカル推論（whisper.cpp/llama.cpp）\*\*でUX基準値を確保、必要時のみクラウドへフォールバック。([GitHub][11])

---

## 4) 価格モデル（試案）

> 目的：**無料でも良体験**、有料は**生産性モジュール/使用量**に応じた価値課金。クラウド依存は**極力オプション**。

| プラン            |        価格(例) | 想定ユーザ      | 含まれる機能                                                                             |
|----------------|-------------:|------------|------------------------------------------------------------------------------------|
| **Free**       |           ¥0 | 個人         | 基本IME/タイプミス修正（ローカルLT）、Anchor Cards（軽量）、月数回の音声入力/翻訳の試用枠。([dev.languagetool.org][5]) |
| **Pro（個人）**    |       ¥900/月 | フリーランス/個人  | 音声入力（ローカル優先/一定分数）、翻訳/テンプレ強化、カスタム辞書同期、Portal高度設定。                                   |
| **Team**       | ¥1,200/ユーザ・月 | スタートアップ/部署 | 共有テンプレ/辞書、チームポリシー、優先サポート、簡易SSO。                                                    |
| **Enterprise** |           見積 | 企業         | オフライン限定構成（エアギャップ）、監査ログ、SLA、専用モデル/辞書。                                               |

**従量課金（追加）**

* **クラウドLLM/翻訳**に切替時のみ**AIクレジット**を消費（ローカルは無料）。
* **音声長時間**は分単位。
* **オンプレ/永久ライセンス**オプション（大口顧客/政府系）。

---

## 5) “Anchor Cards”（Jenny/Jenni AIの発想を発展）

* **文脈駆動**：宛名/件名/曜日/時刻/相手の役職などを軽量特徴量として抽出し、**数個のカード**（「短い挨拶」「丁寧な結び」「確認依頼」「日程提示」など）を
  **候補リスト先頭**に固定表示。
* **入力負担を増やさない**：1キー展開、Tabで差し替え。
* **参考**：Jenni AIの文中オートコンプリート的UX。([Jenni AI][14], [Jenni Help Center][15])

---

## 6) 実装ノート（配布・セキュリティ）

* **Windows**：サードパーティIMEは**デジタル署名必須**、**TSF対応必須**
  。非準拠IMEはブロック対象。設計段階からTSFガイドラインに沿う。([Microsoft Learn][9])
* **macOS**：**InputMethodKit**の規約に従い、**IMKバンドル**としてインストール/有効化。([Apple Developer][2])
* **Linux**：**IBus**のエンジン・コンポーネント仕様に従う。([ibus.github.io][3])
* **Portal**：**Tauri**は各OS標準WebView（WebView2/WKWebView/WebKitGTK）を使うため、軽量・安全。([GitHub][6])

---

## 参考文献（公式・一次情報）

* **Windows IME / TSF**

    * Text Services Framework（概要/リファレンス）・`ITfThreadMgrEx::GetActiveFlags`
      ・IME要件（TSF対応/署名）等。([Microsoft Learn][1])
* **macOS IME**

    * **InputMethodKit**（`IMKServer`含む）公式ドキュメント・IMKサンプル。([Apple Developer][2], [GitHub][10])
* **Linux IME**

    * **IBus**リファレンス（`IBusEngine`/`IBusFactory`）。([ibus.github.io][3], [Tesuji][19])
* **ローカルポータル**

    * **Tauri**（アーキテクチャ：各OSのWebView活用）。([GitHub][6])
    * **Electron**（API/clipboard等、比較用）。([Electron][20])
* **音声入力（ローカル）**

    * **whisper.cpp**（C/C++移植）。([GitHub][11])
* **LLMローカル推論**

    * **llama.cpp**（C/C++、RESTサーバあり）。([GitHub][12])
* **文法/スペル**

    * **LanguageTool**（ローカルHTTPサーバ）。([dev.languagetool.org][5])
* **翻訳**

    * **Marian NMT**・**NLLB-200**（モデルカード/ドキュメント）。([Marian][13], [GitHub][21], [Hugging Face][22])
* **Jenni/Jenny AI（体験の参考情報）**

    * Jenni AI サイト/ヘルプ（オートコンプリート等）。([Jenni AI][14], [Jenni Help Center][15])

---

### 最終提案

* **実行案**：本日から**フェーズA（8週）でmacOS版MVP**を確定。その間にWindowsの署名調達・TSF調査を並行。**フェーズB（6週）でWindows版
  **、**フェーズC（3週）でLinux対応**。
* **なぜこの順序？** IMKは実装が素直で**短期で完成**→ Anchor Cards/Portal含む体験を先に磨ける。Windowsは署名/TSF要件があるため
  **準備が並走**しやすい。([Microsoft Learn][9], [Apple Developer][2])

必要であれば、上記の設計を**要件チケット（Issues）/ガントチャート**に落としてお渡しします。

[1]: https://learn.microsoft.com/en-us/windows/win32/tsf/text-services-framework?utm_source=chatgpt.com "Text Services Framework (Text Services Framework) - learn.microsoft.com"

[2]: https://developer.apple.com/documentation/inputmethodkit?utm_source=chatgpt.com "InputMethodKit | Apple Developer Documentation"

[3]: https://ibus.github.io/docs/ibus-1.5/index.html?utm_source=chatgpt.com "IBus Reference Manual - GitHub Pages"

[4]: https://github.com/fcitx/fcitx5 "GitHub - fcitx/fcitx5: maybe a new fcitx."

[5]: https://dev.languagetool.org/http-server.html?utm_source=chatgpt.com "LanguageTool embedded HTTP Server | dev.languagetool.org"

[6]: https://github.com/tauri-apps/tauri "GitHub - tauri-apps/tauri: Build smaller, faster, and more secure desktop and mobile applications with a web frontend."

[7]: https://github.com/rime/librime "GitHub - rime/librime: Rime Input Method Engine, the core library"

[8]: https://github.com/electron/electron "GitHub - electron/electron: :electron: Build cross-platform desktop apps with JavaScript, HTML, and CSS"

[9]: https://learn.microsoft.com/en-us/windows/apps/design/input/input-method-editors?utm_source=chatgpt.com "Input Method Editors (IME) - Windows apps | Microsoft Learn"

[10]: https://github.com/ensan-hcl/macOS_IMKitSample_2021?utm_source=chatgpt.com "GitHub - ensan-hcl/macOS_IMKitSample_2021: InputMethodKit Sample App ..."

[11]: https://github.com/ggml-org/whisper.cpp "GitHub - ggml-org/whisper.cpp: Port of OpenAI's Whisper model in C/C++"

[12]: https://github.com/ggml-org/llama.cpp "GitHub - ggml-org/llama.cpp: LLM inference in C/C++"

[13]: https://marian-nmt.github.io/?utm_source=chatgpt.com "Marian :: Home"

[14]: https://jenni.ai/?utm_source=chatgpt.com "Jenni AI"

[15]: https://help.jenni.ai/en/collections/12758984-write-with-confidence?utm_source=chatgpt.com "Write with Confidence | Jenni Help Center - help.jenni.ai"

[16]: https://learn.microsoft.com/en-us/windows/win32/api/msctf/nn-msctf-itfthreadmgr?utm_source=chatgpt.com "ITfThreadMgr (msctf.h) - Win32 apps | Microsoft Learn"

[17]: https://learn.microsoft.com/en-us/windows/win32/api/msctf/nf-msctf-itfthreadmgrex-getactiveflags?utm_source=chatgpt.com "ITfThreadMgrEx::GetActiveFlags (msctf.h) - Win32 apps"

[18]: https://ibus.github.io/docs/ibus-1.5/IBusEngine.html?utm_source=chatgpt.com "IBusEngine - GitHub Pages"

[19]: https://tesuji.github.io/ibus/IBusFactory.html?utm_source=chatgpt.com "IBusFactory: IBus Reference Manual"

[20]: https://www.electronjs.org/docs/latest/README "Official Guides | Electron"

[21]: https://github.com/marian-nmt/marian-dev?utm_source=chatgpt.com "GitHub - marian-nmt/marian-dev: Fast Neural Machine Translation in C++ ..."

[22]: https://huggingface.co/facebook/nllb-200-distilled-1.3B?utm_source=chatgpt.com "facebook/nllb-200-distilled-1.3B · Hugging Face"
