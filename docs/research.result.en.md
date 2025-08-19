# Anchor IME Development Framework Selection, Timeline & Pricing Model Proposal

The following presents framework selection, timeline, and pricing model proposals for **Anchor IME** development. As
requested, **official documentation and primary sources** are cited throughout (compiled at the end).

---

## Executive Summary (TL;DR)

* **Recommended Stack (Cross-platform)**
  **Rust core** + **native OS IME APIs** (Windows: **TSF**, macOS: **InputMethodKit**, Linux: **IBus/Fcitx5**) for input
  engine implementation.
  User-facing "uTorrent-style local portal" delivered via **Tauri** (Rust/lightweight WebView).
  Modular feature extensions:

    * **Real-time typo correction**: Custom rules + **LanguageTool** (local HTTP server)
    * **Speech-to-text**: **whisper.cpp** (fully local)
    * **Real-time translation/template generation**: Local LLM (**llama.cpp**) + optional cloud hybrid
    * **Trump card (ref: "Jenny/Jenni AI")**: Context-aware **"Anchor Cards"** (greetings/closings/business text
      templates) presented as card UI in IME suggestion bar
      → **Fast, secure (in-process/local-first), easily extensible**. Windows complies with TSF requirements and signing
      requirements. ([Microsoft Learn][1], [Apple Developer][2], [ibus.github.io][3], [GitHub][4], [dev.languagetool.org][5])

---

## 1) Solution Comparison Matrix

> ★GitHub star counts reflect values displayed on respective repositories as of **August 18, 2025**

| Solution name                                                                | Programming language                                         | Pros                                                                                                                                                                                                                      | Cons                                                                                                                                     | Recommendation (5 pts) | GitHub star (if applicable)                                                                                            | Note                                                                                                              |
|------------------------------------------------------------------------------|--------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------:|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| **Rust Core + OS-Native IME (TSF/IMK/IBus) + Tauri Portal** (Recommended)    | Rust (core/IPC), C++ (TSF wrapper), Swift/Obj-C (IMK bridge) | Each OS official API ensures **compatibility/reliability/minimal latency**. Portal is lightweight **Tauri**. Speech **whisper.cpp**, **LanguageTool**, **llama.cpp** in **fully local configuration** for strong privacy. | OS-specific bridge development costs. Windows distribution requirements (TSF/code signing).                                              |                **5.0** | Tauri **95.6k**★, whisper.cpp **42.4k**★, llama.cpp **85.1k**★, LanguageTool **13.4k**★, IBus **942**★, Fcitx5 **2k**★ | Windows: **TSF** required/signing required. macOS: **IMK** bundle. Linux: **IBus/Fcitx5** engine. ([GitHub][6])   |
| **Rime-based (librime fork) + existing frontends** (Squirrel/ibus-rime etc.) | C++ (librime), Swift (Squirrel), others                      | Rich dictionaries/schemas. Mature morphological conversion and candidate presentation for CJK languages.                                                                                                                  | Core design not necessarily optimized for full IME requirements, heavy AI extension integration. Separate Windows implementation needed. |                **3.8** | librime **3.8k**★, Squirrel **5.1k**★, ibus-rime **805**★                                                              | Good for leveraging existing assets, but limited room for Anchor-specific experience customization. ([GitHub][7]) |
| **Electron Portal + OS-Native IME**                                          | TypeScript/JS (Electron), Rust/C++/Swift (bridges)           | Easy to leverage web development resources. Rich surrounding ecosystem.                                                                                                                                                   | Heavy runtime (memory/startup) and distribution size. Security/auto-update design overhead.                                              |                **3.9** | Electron **118k**★                                                                                                     | Portal-only Electron is viable, but **Tauri is lighter**. ([GitHub][8])                                           |
| **Windows-first MVP (TSF only) + minimal Portal**                            | C++/Rust (TSF), Tauri                                        | Early beta on large Windows market. TSF compliance enables commercial distribution.                                                                                                                                       | macOS/Linux delay. TSF learning curve.                                                                                                   |                **4.2** | (OS API N/A) Portal: Tauri **95.6k**★                                                                                  | Signing/TSF requirement compliance is mandatory. ([Microsoft Learn][9], [GitHub][6])                              |
| **macOS-first MVP (InputMethodKit only) + minimal Portal**                   | Swift/Obj-C (IMK), Tauri                                     | IMK has **relatively straightforward implementation**. Easy to ensure UI quality.                                                                                                                                         | Windows/Linux delay. IMK documentation/samples somewhat dated.                                                                           |                **4.1** | (OS API N/A) Portal: Tauri **95.6k**★                                                                                  | Apple official IMK documentation/samples available. ([Apple Developer][2], [GitHub][10])                          |
| **Pseudo-keyboard/overlay via accessibility APIs** (Not recommended)         | TS/JS (Electron/Tauri)                                       | Can implement without touching IME APIs.                                                                                                                                                                                  | **Not true IME**, compatibility/stability/security issues. Limited in many apps.                                                         |                **1.5** | Electron **118k**★                                                                                                     | Recommended for research only. Difficult to meet commercial-quality input experience. ([GitHub][8])               |

> **Sources (star counts/official APIs)**: Tauri (95.6k), Electron (118k), whisper.cpp (42.4k), llama.cpp (85.1k),
> LanguageTool (13.4k), IBus (942), Fcitx5 (2k), librime (3.8k), Squirrel (5.1k), ibus-rime (805). Current values from
> respective GitHub pages. ([GitHub][6])

---

## 2) Recommended Architecture Key Points

* **IME Core**: Rust for common logic (dictionary, candidate generation, conversion, module pipeline).

    * **Windows**: TSF (Text Services Framework) implementation. Meets IME requirements (**TSF compliance, signing
      required**). Use `ITfThreadMgrEx::GetActiveFlags` to distinguish UWP/desktop. ([Microsoft Learn][9])
    * **macOS**: **InputMethodKit** IMK bundle using `IMKServer` etc. ([Apple Developer][2])
    * **Linux**: **IBus** engine (future Fcitx5 support addition). ([ibus.github.io][3])
* **Local Portal (uTorrent-style)**: **Tauri** (lightweight/secure, leveraging each OS's system WebView). ([GitHub][6])
* **Modules** (process separation/IPC connected)

    * **Typo correction/grammar**: **LanguageTool** local HTTP server (bundled Java
      runtime/configurable). ([dev.languagetool.org][5])
    * **Speech input**: **whisper.cpp** C/C++ bindings. Local streaming recognition. ([GitHub][11])
    * **Translation/template generation/summarization**: Local **llama.cpp** (cloud fallback as needed). NLLB/Marian
      options for low-latency translation. ([GitHub][12], [Marian][13])
* **Trump card (ref: "Jenny/Jenni AI")**: **Anchor Cards**

    * From current input context (subject/recipient/company/language), present **greeting/closing/approval/request
      templates** at top of candidate list as **card UI** (keyboard-only selection/expansion). Inspired by **Jenni AI**'
      s "in-text autocompletion" experience. ([Jenni AI][14], [Jenni Help Center][15])

**Architecture (Conceptual Diagram)**

```

[Key Events] -> [OS IME API (TSF/IMK/IBus)] -> [Rust Core Pipeline]
-> [Speller + Grammar (LanguageTool)]
-> [Translator (NLLB/Marian or LLM)]
-> [LLM (llama.cpp) for templates]
-> [Candidate List + Anchor Cards] -> [Commit to App]
↘︎ [Portal (Tauri) settings/analytics]

```

**Plugin Interface Example**

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

## 3) Detailed Timeline for 3-Person Team (CEO/Junior Dev/Part-time)

> Premise: **macOS-first MVP (8 weeks) → Windows support (+6 weeks) → Linux support (+3 weeks)**
> Workload: CEO=product/specs/QA/procurement, Junior=implementation focus (1.0 FTE),
> Part-time=build/CI/dictionary/tooling (0.5 FTE assumed)

### Phase A: MVP (macOS + Portal + 3 features) — **Weeks 1-8**

| Week | Goal/Deliverable                | Detailed Tasks                                                                           | Assignment                          |
|------|---------------------------------|------------------------------------------------------------------------------------------|-------------------------------------|
| 1    | Requirements freeze, design     | Specs/UX wireframes (including Anchor Cards), IMK/Portal design, data/permissions policy | CEO lead, Dev support               |
| 2    | Skeleton repositories           | Rust core template, IMK bundle template, Tauri template, CI/CD                           | Dev, PT                             |
| 3    | Conversion/candidate foundation | Preedit/candidate display, YAML dictionary loading                                       | Dev                                 |
| 4    | **LanguageTool** integration    | Local HTTP server startup/IPC, dictionary rule addition                                  | Dev, PT ([dev.languagetool.org][5]) |
| 5    | **whisper.cpp** speech          | Hotkey recording→sequential commit, false trigger prevention                             | Dev ([GitHub][11])                  |
| 6    | **Anchor Cards**                | Greeting/closing/reply template (EN/JP) card presentation logic                          | CEO (examples), Dev                 |
| 7    | Portal completion               | Settings (dictionary/shortcuts/translation ON/OFF), logs/diagnostics                     | PT, Dev ([GitHub][6])               |
| 8    | Beta distribution               | Signing/installer, crash collection, QA, user testing                                    | CEO lead                            |

### Phase B: Windows Support (TSF) — **Weeks 9-14**

| Week | Goal/Deliverable                | Detailed Tasks                                                   | Assignment                     |
|------|---------------------------------|------------------------------------------------------------------|--------------------------------|
| 9    | TSF bridge skeleton             | `ITfThreadMgr` activation/context, preedit                       | Dev ([Microsoft Learn][16])    |
| 10   | Candidate UI/switching          | `GetActiveFlags` UWP/desktop branching, existing core connection | Dev ([Microsoft Learn][17])    |
| 11   | Signing/distribution prep       | Code signing certificate, installer                              | CEO, PT ([Microsoft Learn][9]) |
| 12   | LanguageTool/speech integration | Port modules implemented on macOS                                | Dev                            |
| 13   | QA/optimization                 | Latency/IME switching/shortcut verification                      | All                            |
| 14   | Beta distribution               | Internal/limited user deployment                                 | CEO                            |

### Phase C: Linux Support (IBus) — **Weeks 15-17**

| Week | Goal/Deliverable  | Detailed Tasks                                       | Assignment                 |
|------|-------------------|------------------------------------------------------|----------------------------|
| 15   | IBus skeleton     | `IBusEngine` implementation, settings UI integration | Dev ([ibus.github.io][18]) |
| 16   | Verification      | Wayland/X11, major app operation confirmation        | Dev, PT                    |
| 17   | Beta distribution | Packaging (.deb/.rpm/AppImage)                       | PT                         |

**Risks/Mitigation**

* **TSF learning curve** → Complete MVP with IMK first, apply TSF from **minimum requirements**
  progressively. ([Microsoft Learn][1])
* **Signing/distribution** → Plan and procure early to meet Windows IME requirements (**TSF compliance, signing required
  **). ([Microsoft Learn][9])
* **Speech/translation latency** → Secure UX baseline with **local inference (whisper.cpp/llama.cpp)** first, cloud
  fallback only when necessary. ([GitHub][11])

---

## 4) Pricing Model (Draft)

> Objective: **Good experience even free**, paid for **productivity modules/usage** value-based pricing. Cloud
> dependency **largely optional**.

| Plan                 | Price (example)   | Target Users          | Included Features                                                                                                                               |
|----------------------|-------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| **Free**             | ¥0                | Individual            | Basic IME/typo correction (local LT), Anchor Cards (lightweight), monthly trial quota for speech input/translation. ([dev.languagetool.org][5]) |
| **Pro (Individual)** | ¥900/month        | Freelancer/Individual | Speech input (local-first/limited minutes), enhanced translation/templates, custom dictionary sync, advanced Portal settings.                   |
| **Team**             | ¥1,200/user/month | Startup/Department    | Shared templates/dictionaries, team policies, priority support, basic SSO.                                                                      |
| **Enterprise**       | Quote             | Corporation           | Offline-only configuration (air-gapped), audit logs, SLA, dedicated models/dictionaries.                                                        |

**Usage-based charges (additional)**

* **AI credits** consumed only when switching to **cloud LLM/translation** (local is free).
* **Extended speech** per-minute billing.
* **On-premise/perpetual license** option (large customers/government).

---

## 5) "Anchor Cards" (Evolving Jenny/Jenni AI Concept)

* **Context-driven**: Extract recipient/subject/day/time/role as lightweight features, display **several cards** ("brief
  greeting", "polite closing", "confirmation request", "schedule proposal") **fixed at top of candidate list**.
* **No added input burden**: One-key expansion, Tab for replacement.
* **Reference**: Jenni AI's in-text autocompletion UX. ([Jenni AI][14], [Jenni Help Center][15])

---

## 6) Implementation Notes (Distribution, Security)

* **Windows**: Third-party IMEs require **digital signature**, **TSF compliance**. Non-compliant IMEs are blocked.
  Design according to TSF guidelines from the start. ([Microsoft Learn][9])
* **macOS**: Follow **InputMethodKit** regulations, install/enable as **IMK bundle**. ([Apple Developer][2])
* **Linux**: Follow **IBus** engine/component specifications. ([ibus.github.io][3])
* **Portal**: **Tauri** uses each OS's standard WebView (WebView2/WKWebView/WebKitGTK), ensuring lightweight and secure
  operation. ([GitHub][6])

---

## References (Official/Primary Sources)

* **Windows IME / TSF**
    * Text Services Framework (overview/reference), `ITfThreadMgrEx::GetActiveFlags`, IME requirements (TSF
      compliance/signing), etc. ([Microsoft Learn][1])
* **macOS IME**
    * **InputMethodKit** (including `IMKServer`) official documentation, IMK
      samples. ([Apple Developer][2], [GitHub][10])
* **Linux IME**
    * **IBus** reference (`IBusEngine`/`IBusFactory`). ([ibus.github.io][3], [Tesuji][19])
* **Local Portal**
    * **Tauri** (architecture: leveraging each OS's WebView). ([GitHub][6])
    * **Electron** (API/clipboard etc., for comparison). ([Electron][20])
* **Speech Input (Local)**
    * **whisper.cpp** (C/C++ port). ([GitHub][11])
* **Local LLM Inference**
    * **llama.cpp** (C/C++, with REST server). ([GitHub][12])
* **Grammar/Spelling**
    * **LanguageTool** (local HTTP server). ([dev.languagetool.org][5])
* **Translation**
    * **Marian NMT**, **NLLB-200** (model cards/documentation). ([Marian][13], [GitHub][21], [Hugging Face][22])
* **Jenni/Jenny AI (Experience Reference)**
    * Jenni AI site/help (autocompletion etc.). ([Jenni AI][14], [Jenni Help Center][15])

---

### Final Recommendation

* **Action Plan**: Start **Phase A (8 weeks) macOS MVP** today. Parallel Windows signing procurement and TSF research. *
  *Phase B (6 weeks) Windows version**, **Phase C (3 weeks) Linux support**.
* **Why this order?** IMK implementation is straightforward for **quick completion** → Polish Anchor Cards/Portal
  experience first. Windows has signing/TSF requirements making **parallel preparation**
  efficient. ([Microsoft Learn][9], [Apple Developer][2])

If needed, I can break down the above design into **requirement tickets (Issues)/Gantt charts** for delivery.

[References 1-22 remain the same as in the original]
