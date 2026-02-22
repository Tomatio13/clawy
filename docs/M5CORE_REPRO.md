# Clawy を M5Stack Core で再現する手順（Claude 連動込み）

この手順は、上流の Clawy 本体が更新された後でも、**M5Stack Core 向けの表示調整と Claude 連動設定**を再適用できるようにしたものです。

## 対象
- デバイス: `M5Stack Core`（FQBN: `m5stack:esp32:m5stack_core`）
- OS: Linux/macOS
- CLI: `arduino-cli`

## 含まれる変更
このドキュメントで再適用する変更は以下です。
- Core 向けの表示最適化（向き、スケール、余白、文字サイズ）
- Core で黒画面化しないための挙動調整
- `hooks` の通信先解決改善（`CLAWY_HOST` / `~/.clawy/host` 優先）

変更ファイル:
- `firmware/clawy/clawy.ino`
- `firmware/clawy/display.h`
- `hooks/send-status.sh`
- `hooks/permission-listener.sh`
- `README.md`

再適用パッチ:
- `docs/patches/m5core-claude-integration.patch`

## 1. バックアップ
```bash
TS=$(date +%Y%m%d-%H%M%S)
BDIR="$HOME/.clawy-backups/$TS"
mkdir -p "$BDIR"
[ -f "$HOME/.claude/settings.json" ] && cp -a "$HOME/.claude/settings.json" "$BDIR/settings.json.bak"
[ -f "$HOME/.zshrc" ] && cp -a "$HOME/.zshrc" "$BDIR/.zshrc.bak"
[ -f "$HOME/.bashrc" ] && cp -a "$HOME/.bashrc" "$BDIR/.bashrc.bak"
[ -d "$HOME/.clawy" ] && cp -a "$HOME/.clawy" "$BDIR/.clawy.bak"
echo "backup: $BDIR"
```

## 2. 上流更新後にパッチ再適用
リポジトリ直下で実行:
```bash
git apply --3way docs/patches/m5core-claude-integration.patch
```

衝突した場合は対象ファイルを手動で解決してください（上の「変更ファイル」参照）。

## 3. Arduino コアとライブラリ準備
```bash
./arduino-cli core update-index \
  --additional-urls https://static-cdn.m5stack.com/resource/arduino/package_m5stack_index.json

./arduino-cli core install m5stack:esp32 \
  --additional-urls https://static-cdn.m5stack.com/resource/arduino/package_m5stack_index.json

./arduino-cli lib install "M5Unified" "M5GFX"
```

## 4. Core 向けビルド
```bash
./arduino-cli compile -b m5stack:esp32:m5stack_core firmware/clawy
```

## 5. Core へ書き込み
```bash
./arduino-cli upload -p /dev/ttyUSB0 -b m5stack:esp32:m5stack_core firmware/clawy
```

ポートは環境に合わせて変更:
- Linux 例: `/dev/ttyUSB0`
- macOS 例: `/dev/cu.usbserial-*`

## 6. Claude 連動設定（hooks）
```bash
./install.sh
```

`install.sh` は以下を実施します。
- `~/.clawy/hooks/*` 配置
- `~/.claude/settings.json` に hooks 追加
- `clawy()` 関数の追加（シェル設定）

## 7. mDNS 失敗時の固定IP設定（重要）
`clawy.local` が解決できない環境では固定IPを設定します。

```bash
echo "<M5のIP>" > ~/.clawy/host
# 例:
echo "10.108.170.171" > ~/.clawy/host
```

hooks の解決優先順位:
1. `CLAWY_HOST` 環境変数
2. `~/.clawy/host`
3. キャッシュ (`/tmp/clawy-ip-$USER`)
4. `clawy.local`

## 8. 動作確認
### 8-1. デバイス到達確認
```bash
python3 - <<'PY'
import socket
host='10.108.170.171'  # 必要に応じて変更
for p in (7800,7801):
    try:
        s=socket.create_connection((host,p),timeout=2)
        print('ok',p)
        s.close()
    except Exception as e:
        print('ng',p,e)
PY
```

### 8-2. hooks 単体テスト
```bash
CLAWY=1 ~/.clawy/hooks/send-status.sh READY
CLAWY=1 ~/.clawy/hooks/send-status.sh "TOOL:Running" "connectivity test"
```

### 8-3. Claude 連動実運用
新しいターミナルで:
```bash
source ~/.zshrc  # bashなら source ~/.bashrc
clawy
```

## 9. よくあるハマりどころ
- 何も連動しない:
  - `CLAWY=1` が付いていないセッションで起動している
  - `clawy.local` が解決できていない（`~/.clawy/host` を設定）
- Core で画面が黒くなる:
  - Core 向けの sleep 無効化パッチが外れている可能性
- 反映されない:
  - 既存 Claude セッションを閉じずに再テストしている（再起動する）

## 10. ロールバック
```bash
# backup ディレクトリは 1. で出力されたパス
BDIR="<backup-dir>"
[ -f "$BDIR/settings.json.bak" ] && cp -a "$BDIR/settings.json.bak" ~/.claude/settings.json
[ -f "$BDIR/.zshrc.bak" ] && cp -a "$BDIR/.zshrc.bak" ~/.zshrc
[ -f "$BDIR/.bashrc.bak" ] && cp -a "$BDIR/.bashrc.bak" ~/.bashrc
[ -d "$BDIR/.clawy.bak" ] && rm -rf ~/.clawy && cp -a "$BDIR/.clawy.bak" ~/.clawy
```
