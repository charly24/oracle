---
name: oracle
description: Use the @steipete/oracle CLI to bundle a prompt plus the right files and get a second-model review (API or browser) for debugging, refactors, design checks, or cross-validation.
---

# Oracle (CLI) — best use

Oracle bundles your prompt + selected files into one “one-shot” request so another model can answer with real repo context (API or browser automation). Treat outputs as advisory: verify against the codebase + tests.

## Main use case (browser, GPT‑5.5 Pro)

Default workflow here: `--engine browser` with GPT‑5.5 Pro in ChatGPT. This is the “human in the loop” path: it can take ~10 minutes to ~1 hour; expect a stored session you can reattach to.

Recommended defaults:

- Engine: browser (`--engine browser`)
- Model: GPT‑5.5 Pro (use `--model gpt-5.5-pro`; add `--browser-model-strategy select` when the user explicitly asks for 5.5/Pro)
- Attachments: directories/globs + excludes; avoid secrets.

## Golden path (fast + reliable)

1. Pick a tight file set (fewest files that still contain the truth).
2. Preview what you’re about to send (`--dry-run` + `--files-report` when needed).
3. Run in browser mode for the usual GPT‑5.5 Pro ChatGPT workflow; use API only when you explicitly want it.
4. If the run detaches/timeouts: reattach to the stored session (don’t re-run).

## Commands (preferred)

- Show help (once/session):
  - `npx -y @steipete/oracle --help`

- Preview (no tokens):
  - `npx -y @steipete/oracle --dry-run summary -p "<task>" --file "src/**" --file "!**/*.test.*"`
  - `npx -y @steipete/oracle --dry-run full -p "<task>" --file "src/**"`

- Token/cost sanity:
  - `npx -y @steipete/oracle --dry-run summary --files-report -p "<task>" --file "src/**"`

- Browser run (main path; long-running is normal):
  - `npx -y @steipete/oracle --engine browser --model gpt-5.5-pro --browser-model-strategy select -p "<task>" --file "src/**"`

- Manual paste fallback (assemble bundle, copy to clipboard):
  - `npx -y @steipete/oracle --render --copy -p "<task>" --file "src/**"`
  - Note: `--copy` is a hidden alias for `--copy-markdown`.

## Attaching files (`--file`)

`--file` accepts files, directories, and globs. You can pass it multiple times; entries can be comma-separated.

- Include:
  - `--file "src/**"` (directory glob)
  - `--file src/index.ts` (literal file)
  - `--file docs --file README.md` (literal directory + file)

- Exclude (prefix with `!`):
  - `--file "src/**" --file "!src/**/*.test.ts" --file "!**/*.snap"`

- Defaults (important behavior from the implementation):
  - Default-ignored dirs: `node_modules`, `dist`, `coverage`, `.git`, `.turbo`, `.next`, `build`, `tmp` (skipped unless you explicitly pass them as literal dirs/files).
  - Honors `.gitignore` when expanding globs.
  - Does not follow symlinks (glob expansion uses `followSymbolicLinks: false`).
  - Dotfiles are filtered unless you explicitly opt in with a pattern that includes a dot-segment (e.g. `--file ".github/**"`).
  - Default cap: files > 1 MB are rejected unless you raise `ORACLE_MAX_FILE_SIZE_BYTES` or `maxFileSizeBytes` in `~/.oracle/config.json`.

## Budget + observability

- Target: keep total input under ~196k tokens.
- Use `--files-report` (and/or `--dry-run json`) to spot the token hogs before spending.
- If you need hidden/advanced knobs: `npx -y @steipete/oracle --help --verbose`.

## Engines (API vs browser)

- Auto-pick: uses `api` when `OPENAI_API_KEY` is set, otherwise `browser`.
- Browser engine supports GPT + Gemini only; use `--engine api` for Claude/Grok/Codex or multi-model runs.
- **API runs require explicit user consent** before starting because they incur usage costs.
- Browser attachments:
  - `--browser-attachments auto|never|always` (auto pastes inline up to ~60k chars then uploads).
- Remote browser host (signed-in machine runs automation):
  - Host: `oracle serve --host 0.0.0.0 --port 9473 --token <secret>`
  - Client: `oracle --engine browser --remote-host <host:port> --remote-token <secret> -p "<task>" --file "src/**"`

## Sessions + slugs (don’t lose work)

- Stored under `~/.oracle/sessions` (override with `ORACLE_HOME_DIR`).
- Runs may detach or take a long time (browser + GPT‑5.5 Pro often does). If the CLI times out: don’t re-run; reattach.
  - List: `oracle status --hours 72`
  - Attach: `oracle session <id> --render`
- Use `--slug "<3-5 words>"` to keep session IDs readable.
- Duplicate prompt guard exists; use `--force` only when you truly want a fresh run.

## Prompt template (high signal)

Oracle starts with **zero** project knowledge. Assume the model cannot infer your stack, build tooling, conventions, or “obvious” paths. Include:

- Project briefing (stack + build/test commands + platform constraints).
- “Where things live” (key directories, entrypoints, config files, dependency boundaries).
- Exact question + what you tried + the error text (verbatim).
- Constraints (“don’t change X”, “must keep public API”, “perf budget”, etc).
- Desired output (“return patch plan + tests”, “list risky assumptions”, “give 3 options with tradeoffs”).

### “Exhaustive prompt” pattern (for later restoration)

When you know this will be a long investigation, write a prompt that can stand alone later:

- Top: 6–30 sentence project briefing + current goal.
- Middle: concrete repro steps + exact errors + what you already tried.
- Bottom: attach _all_ context files needed so a fresh model can fully understand (entrypoints, configs, key modules, docs).

If you need to reproduce the same context later, re-run with the same prompt + `--file …` set (Oracle runs are one-shot; the model doesn’t remember prior runs).

## Safety

- Don’t attach secrets by default (`.env`, key files, auth tokens). Redact aggressively; share only what’s required.
- Prefer “just enough context”: fewer files + better prompt beats whole-repo dumps.

---

# MCP 経由（`mcp__oracle__consult`）の実証済み安定レシピ

セカンドオピニオン・重厚な設計/標準レビューを MCP ツール（ChatGPT ブラウザ自動化）で回すときの**実証済み安定レシピ**。過去にブラウザ抽出が `I`（1トークン）だけ返って失敗していたが、下記構成で全文抽出が安定して取れる（実績: GPT-5.5 Pro / extended で 11分・入力33k・出力2.7k を正常取得）。

## 安定パラメータ（このまま使う）

- `engine: "browser"` ＋ `preset: "chatgpt-pro-heavy"`（ChatGPT Pro ブラウザ＋現行 Pro モデル別名＋Extended thinking を一括選択）
- `browserAttachments: "always"` — **実ファイルアップロード**で渡す（巨大プロンプトのインライン貼り付けは抽出を壊しやすい）
- 添付は**自己完結の単一 .md ファイル**にまとめる（文脈＋設問＋レビュー対象を1ファイルに連結 → `files=1`）。複数ファイル bundle より単一ファイルの方が安定
- `prompt` には「依頼の要点＋出力フォーマット（番号付きの章立て）」を再掲する（詳細は添付ファイル側に持たせる）
- `browserThinkingTime: "extended"`
- `slug` を付ける（後で `mcp__oracle__sessions` で参照できる）

## 運用上の注意

- **`browserModelStrategy:"select"` を preset と併用しない**。`chatgpt-pro-heavy` preset 単独（modelStrategy は既定の `current`）が安定。`select` を明示すると Pro の Extended thinking チップ取得に失敗し `Thinking time: chip not found for pro (requested Extended); refusing to submit` で即死することがある（2026-06-16 実測。preset のみに落として再実行で成功）。実績: 単一 .md 添付（docs 42ファイル連結・555KB・~174k tokens）を `preset:chatgpt-pro-heavy` ＋ `browserAttachments:"always"` ＋ `browserThinkingTime:"extended"` で 8分27秒・全文取得。
- **MCP の返却 `output` が末尾だけに切れることがある**（§8-10 のみ等）。全文は `mcp__oracle__sessions({id, detail:true})` の `log` フィールドに入っている（`Answer:` 以降）。ChatGPT 会話にも残る。
- ブラウザ自動化は数分〜十数分かかる。`dryRun:true` で設定を確認してから本実行してよい。
- 万一抽出がスタブ（`I` 等）で返っても、**回答全文は ChatGPT プロジェクト側の会話に残る**。`mcp__oracle__sessions` で session を引くか、ユーザーに ChatGPT で開いてもらい貼り戻す。
- 「2巡」運用が有効: ①ベースライン（対象だけ）→ ②文脈付き（当社の体制・現在地・狙い・確定方針を冒頭に付した自己完結パケット）。文脈付きの方が右サイズ判断・過不足指摘の精度が上がる。
- パケットは `tmp/oracle/` に置き、結果も `tmp/oracle/<slug>-result.md` として保存して証跡化する（tmp はコミットしない）。

# 知見の蓄積（Self-Improvement）

Oracle はブラウザ自動化ゆえ不安定なことがある。**うまくいったパターンを見つけたら、このスキルに追記して更新する**（ユーザーの CLAUDE.md には書かない。ここが正）。

- 候補（運用中・実証でき次第レシピ化）: **zip にまとめたドキュメント群を添付してレビューさせるパターン** — 安定動作が確認できた回で、zip の作り方（構成・サイズ・目次の持たせ方）と prompt の組み方をここに追記する。
