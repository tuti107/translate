# translate プロジェクト 申し送り

**作成日:** 2026-05-05
**作成セッション:** Windows ネイティブ Claude Code (`c:\Users\tsuch\dev\translate`)
**次セッションの想定環境:** WSL Ubuntu (aarch64)、ユーザー `tuti107`、プロジェクト位置 `~/dev/translate`

---

## 0. 結論先出し（次セッションへの即効リスト）

1. プロジェクトは **WSL ext4 上の `~/dev/translate`** に作る（`/mnt/c/...` は I/O が 10〜20 倍遅いので避ける）
2. Claude Code は **VS Code Remote-WSL 経由 or WSL シェル** から起動する。Windows ネイティブから起動すると後述の RTK フックが効かない
3. 初回起動後にやること:
   - `rtk init --show` で 4 項目 `[ok]` を確認
   - 本ドキュメントを `~/dev/translate/HANDOFF.md` にコピー（現状は `/mnt/c/Users/tsuch/dev/translate/HANDOFF.md` にある）
   - Windows 側のメモリ（後述）を WSL 側プロジェクトメモリへ複製

---

## 1. 環境前提

| 項目 | 値 |
|---|---|
| ホスト OS | Windows 11 Home (ARM64) |
| WSL ディストリ | Ubuntu (WSL2, aarch64) |
| WSL ユーザー | `tuti107` |
| WSL ホーム | `/home/tuti107` |
| Windows ユーザー | `tsuch` |
| 旧プロジェクト位置 | `c:\Users\tsuch\dev\translate`（空・破棄予定） |
| **新プロジェクト位置** | `~/dev/translate`（WSL ext4 内、未作成） |

---

## 2. RTK (Rust Token Killer) 設置状況

WSL 側に v0.38.0 を導入済み。**Windows ネイティブ側には未導入**（ネイティブではフックが動作しないため）。

### バイナリ・PATH

| 項目 | 状態 |
|---|---|
| `rtk --version` | ✅ `rtk 0.38.0` |
| インストール先 | ✅ `/home/tuti107/.local/bin/rtk`（aarch64-unknown-linux-gnu） |
| PATH 設定 | ✅ `~/.bashrc` に `export PATH="$HOME/.local/bin:$PATH"` を末尾追記済み |
| バイナリ整合性 | ✅ tarball SHA256 = `2e171f1d1c76086bb447e372d9332869d2cd3cc106c08c6e5fbd102b12e91ad9`（公式 checksums.txt と一致確認済み） |

### Claude Code 連携（WSL 側）

| 項目 | 状態 |
|---|---|
| `~/.claude/settings.json` | ✅ PreToolUse / Bash matcher → `rtk hook claude` 登録 |
| `~/.claude/RTK.md` | ✅ slim mode (10 行) 配置 |
| `~/.claude/CLAUDE.md` | ✅ `@RTK.md` 参照あり（**ただし、後述のトークン節約ルールは未追記**） |
| `~/.config/rtk/filters.toml` | ✅ ユーザーカスタムフィルタ用テンプレート配置 |
| `rtk init --show` | ✅ Hook / RTK.md / CLAUDE.md / settings.json の 4 項目すべて `[ok]` |

### 動作確認方法（次セッションで実行）

```bash
which rtk                       # → /home/tuti107/.local/bin/rtk
rtk init --show                 # 4 項目 [ok] を確認
git status                      # フック経由で RTK が走る
rtk gain                        # 削減統計が記録されていれば OK
```

---

## 3. トークン効率化ルール（最重要）

ユーザーは Claude Code のトークン消費を **第一級の関心事** として扱う。"RTK 動作検証 & トークン最適化提案" レポート（2026-05-03 作成）を共有しており、その指針に従うことが期待されている。

### 適用済みのルール（Windows 側）

`c:\Users\tsuch\.claude\CLAUDE.md` に下記を記載済み（提案 G「ツール選択」を実装したもの）:

| 目的 | 推奨 | 回避 |
|---|---|---|
| ファイル名で検索 | `Glob` | `find -name`, `ls -R`, `dir /s` |
| 内容を検索 | `Grep`（既定で `output_mode: files_with_matches`） | `grep -r`, `Select-String -Recurse` |
| 既知ファイルを読む | `Read`（大ファイルは `offset`/`limit`） | `cat`, `Get-Content`, `head`, `tail` |
| ディレクトリ構造 | `Glob "**/*.{ext}"` | `tree`, `ls -laR`, `Get-ChildItem -Recurse` |
| 編集 | `Edit` | `sed`, `awk`, here-doc 上書き |
| 作成 | `Write` | `echo > file`, `Set-Content` |

加えて: **大規模探索（>3 クエリ）は Explore サブエージェントへ委譲**（親コンテキストに探索結果が載らない最大の節約手段）。

### **要対応:** WSL 側の `~/.claude/CLAUDE.md` 拡張

現状 WSL 側の `~/.claude/CLAUDE.md` は `@RTK.md` 参照のみ。次セッションで上記ルールを WSL 側にも反映するのが望ましい。やり方は 2 通り:

- **A 案（推奨）:** Windows 側の `c:\Users\tsuch\.claude\CLAUDE.md` の中身を WSL 側 `~/.claude/CLAUDE.md` に追記（`@RTK.md` 行は残す）
- **B 案:** ルールを別ファイル（例: `~/.claude/TOOL_RULES.md`）に切り出し、`~/.claude/CLAUDE.md` から `@TOOL_RULES.md` で参照

---

## 4. メモリ状態

### Windows 側（現セッション）

`c:\Users\tsuch\.claude\projects\c--Users-tsuch-dev-translate\memory\` に下記を保存:

- `MEMORY.md` — インデックス
- `feedback_token_efficiency.md` — 「Read/Grep/Glob を Bash より優先、Explore サブエージェント活用」が project-level feedback として記録

### WSL 側（次セッション）

プロジェクトパスが変わる（`~/dev/translate`）ため、Claude Code は **新しい hashed ディレクトリ** を作る:

```
~/.claude/projects/-home-tuti107-dev-translate/memory/
```

ここに上記 2 ファイルを複製しておくと、次セッションが同じ feedback を引き継げる。**コマンド例（WSL シェルで実行）:**

```bash
SRC=/mnt/c/Users/tsuch/.claude/projects/c--Users-tsuch-dev-translate/memory
DST=$HOME/.claude/projects/-home-tuti107-dev-translate/memory
mkdir -p "$DST"
cp "$SRC/MEMORY.md" "$SRC/feedback_token_efficiency.md" "$DST/"
```

> 注: WSL 内の Claude Code が初回起動時に `-home-tuti107-dev-translate` ディレクトリを実際にどう命名するかは、起動時の `cwd` から決まる。命名規則が違っていたら `ls ~/.claude/projects/` で実際のディレクトリ名を確認してから `mv`。

---

## 5. ユーザー情報

- メール: `tsuchida.yasuhiro107@gmail.com`
- 開発スタイル: トークン効率を強く意識。Linux マシン（別ホスト `tuti107`）でも RTK 運用中。Windows + WSL は今回新規立ち上げ
- 想定言語: 日本語（プロジェクト名 "translate" から推測されるが、用途は未確定）

---

## 6. 「translate」プロジェクトの内容について

**現時点で中身は未定**。ディレクトリは Windows 側で空のまま作成されている。新セッションでユーザーから具体要件を引き出してから、

- プロジェクト直下に `CLAUDE.md` を新規作成（提案 A）
- 主要構造・依存・ビルド/実行コマンドを記載
- これにより以後のセッションでの探索コストが激減

を実施するのが本筋。

---

## 7. やってはいけないこと（既出の落とし穴）

1. `wsl.exe -d Ubuntu -u tuti107 -- bash -lc '...'` で `$(...)` や `awk '{print $1}'` を使うと **クォートが壊れて変数が空になる**。複雑なシェル処理は `cat > /tmp/foo.sh << "EOF" ... EOF; chmod +x /tmp/foo.sh; /tmp/foo.sh` のスクリプトファイル経由で実行する
2. `rtk init -g` を素で実行すると settings.json 上書き確認プロンプトが出て **非対話モードでは N に倒れる**。必ず `--auto-patch` を付ける
3. **Windows ネイティブ Claude Code から RTK は動かない**。フックは WSL 側 `~/.claude/settings.json` に登録されているため

---

## 8. 参考リンク

- RTK プロジェクト: https://github.com/rtk-ai/rtk
- RTK v0.38.0 リリース: https://github.com/rtk-ai/rtk/releases/tag/v0.38.0
- Claude Code 連携ドキュメント: https://deepwiki.com/rtk-ai/rtk/2.3-claude-code-integration
- 元提案ドキュメント: ユーザー提供の "RTK 動作検証 & トークン最適化提案"（2026-05-03）

---

## 9. 次セッション開始時のチェックリスト

- [ ] WSL 内で Claude Code を起動できているか（`pwd` が `/home/tuti107/...` で始まる）
- [ ] `rtk init --show` 4 項目 `[ok]`
- [ ] 本ドキュメントが `~/dev/translate/HANDOFF.md` に存在
- [ ] WSL 側プロジェクトメモリに `MEMORY.md` と `feedback_token_efficiency.md` を複製済み
- [ ] WSL 側 `~/.claude/CLAUDE.md` にツール選択ルールを追記済み
- [ ] `git status` を 1 度実行 → `rtk gain` で記録されていることを確認
- [ ] ユーザーから "translate" プロジェクトの具体要件を聞き、プロジェクト直下 `CLAUDE.md` を作成
