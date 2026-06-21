# create-colonies (packwiz modpack)

udondon の Minecraft サーバー (NeoForge 1.21.1) 用 mod 管理リポジトリ。
packwiz でビルドし GitHub Pages から配信、サーバー起動時に自動同期されます。

- Pack URL: <https://udondon1478.github.io/create-colonies/pack.toml>
- 対象サーバー: noraru-a1 上の Pelican volume `ad3ed947-40f9-4b39-92d0-cea60c9fbe20`
- MC: 1.21.1 / NeoForge: 21.1.230

---

## 仕組み

サーバー起動 → `run.sh` が `packwiz-installer-bootstrap.jar` を実行 → このリポジトリの `pack.toml` を取得 → 差分のみ DL/更新/削除 → NeoForge が起動。

```sh
# run.sh の該当部分
java -jar packwiz-installer-bootstrap.jar -g -s server "$PACK_URL"
```

- `-s server`: side が `server` または `both` の mod のみインストール (client-only は無視)
- 既存の `mods/*.jar` のうち pack に含まれるものはハッシュで検証、不一致なら再 DL
- pack の管理外の jar (例: 手動で置いたもの) は **触らない**

---

## 運用ワークフロー

### 編集場所
サーバー (`noraru-a1`) 上の `~/packwiz-udondon` が作業ディレクトリ。SSH で入って操作する想定。

ローカル PC で作業したい場合は `git clone` してから packwiz をインストール (要 Go ビルド、ARM 以外なら nightly.link)。

### mod を追加

```sh
cd ~/packwiz-udondon

# Modrinth (推奨、API key 不要)
packwiz mr add <slug-or-url>
# 例: packwiz mr add jade

# CurseForge (Modrinth に無い場合)
packwiz cf add <slug-or-url>
# 例: packwiz cf add ftb-chunks

# 直接 URL から (Modrinth/CF どちらにも無い場合)
packwiz url add <url>

# まとめて反映
packwiz refresh
git add -A && git commit -m "add: <mod-name>" && git push
```

→ 次回サーバー起動時に自動 DL。

### サーバーフォルダに手動で入れた mod を pack に取り込む

サーバーの `mods/` に jar を手動コピーした場合、それは pack 管理外。pack に組み込みたいときは:

```sh
cd ~/packwiz-udondon

# 該当 jar を pack の mods/ にコピー (サーバー volume から)
VOL=/var/lib/pelican/volumes/ad3ed947-40f9-4b39-92d0-cea60c9fbe20
sudo cp $VOL/mods/<filename>.jar mods/
sudo chown ubuntu:ubuntu mods/<filename>.jar

# Modrinth 由来なら一括検出: hash で Modrinth に問い合わせて自動追加 (このリポジトリ初期化時と同じ方式)
# CurseForge 由来ならこちらで一括検出 (fingerprint で照合)
packwiz cf detect -y

# どちらにも無い場合、jar をリポジトリに残して url add
# (GitHub Pages の raw URL を使う: https://udondon1478.github.io/create-colonies/mods/<filename>.jar)
packwiz url add "https://udondon1478.github.io/create-colonies/mods/<filename>.jar"

# 反映
packwiz refresh
git add -A && git commit -m "import: <mod-name>" && git push
```

検出後、サーバー側の手動コピーは `rm` してよい (次回起動で bootstrap が pack から再 DL する)。

### mod を削除

```sh
cd ~/packwiz-udondon
packwiz remove <name>
packwiz refresh
git add -A && git commit -m "remove: <name>" && git push
```

→ 次回起動時にサーバーの `mods/` から自動削除。

### mod のバージョンを更新

```sh
cd ~/packwiz-udondon
packwiz update <name>      # 個別
packwiz update --all       # 一括
packwiz refresh
git add -A && git commit -m "update: ..." && git push
```

特定バージョンに固定したい場合は `packwiz pin <name>` で更新対象から除外。

### バージョン固定 / 固定解除

```sh
packwiz pin <name>     # 更新対象から外す
packwiz unpin <name>   # 更新対象に戻す
```

---

## 注意事項

- **クライアント mod の扱い**: このリポジトリは server-side 用。クライアント配布は別途 AutoModPack を使用。client-only mod を pack に追加する場合は `side = "client"` を `mods/<name>.pw.toml` に手書きで設定 (bootstrap が `-s server` で実行されるため、サーバーには入らない)。
- **CurseForge API key**: 通常不要 (CDN URL が public)。新規 mod 検索で API rate limit に当たった場合のみ環境変数 `CFCORE_API_KEY` 設定を検討。
- **transitive dependency**: `packwiz mr add` が依存 mod を自動追加することがある (geckolib 等)。不要なら `packwiz remove` で削除可。
- **トラブル時**: `run.sh` のバックアップが `<volume>/run.sh.bak.before-packwiz` にある。差し戻す場合は `sudo cp` で復元。

---

## 関連ファイル

| パス | 役割 |
|---|---|
| `~/packwiz-udondon/` | このリポジトリの clone (サーバー上の作業ディレクトリ) |
| `~/bin/packwiz` | packwiz CLI バイナリ (ARM64 source build) |
| `<volume>/run.sh` | bootstrap を呼ぶ起動スクリプト |
| `<volume>/packwiz-installer-bootstrap.jar` | 同期エンジン本体 (Java) |
| `<volume>/packwiz.json` | bootstrap が管理する manifest (自動生成、編集不要) |
