# Docker Compose — 起動手順と今回の実装の詳細

このドキュメントは、プロジェクトで追加した Docker Compose 構成の起動手順と、今回の実装内容（php, postgres, nginx）について誰でも簡単に理解できるようにまとめたものです。

## ファイル構成（今回追加／変更された主なファイル）
- `docker-compose.yml` — プロジェクトルートにある Compose 定義（サービス: `php`, `db`, `web`）。
- `docker/nginx/default.conf` — nginx の最小設定（`/var/www/html` をドキュメントルート、PHP を `php:9000` に渡す）。
- `index.php` — 動作確認用の簡易 PHP ページ（存在しない場合は 403 が出るため確認用に追加）。

## 目的
- 開発に必要となる 3 つの基本コンテナ（PHP-FPM, PostgreSQL, Nginx）を定義し、次の Issue で機能を追加できる状態にする。

## 主要サービス概要
- php: `php:8.4-fpm`（Drupal 11.2 を使うため 8.4 系に変更済み）。ワーキングディレクトリ `/var/www/html` にプロジェクトをマウントします。
- db: `postgres:15`。環境変数で初期 DB とユーザをセットし、永続化のため `db_data` ボリュームを使用します。
- web: `nginx:1.25-alpine`。ホストの `8080` をコンテナの `80` に公開し、`docker/nginx/default.conf` を読み込んで PHP-FPM に渡します。

全サービスは `streak_net` というブリッジネットワークに属します。

## 起動手順（ローカルでの最小確認）
1. Docker と Docker Compose がインストールされていることを確認してください。
2. プロジェクトルートに移動します。

```bash
cd /path/to/STREAK.s
```

3. コンテナをビルドして起動します（バックグラウンド）：

```bash
docker compose up -d --build
```

4. 起動状態を確認します：

```bash
docker compose ps
```

5. nginx のログや各サービスのログを確認する場合：

```bash
docker compose logs -f web
docker compose logs -f php
docker compose logs -f db
```

6. HTTP 動作確認（ブラウザで http://localhost:8080 を開くか、curl を使う）：

```bash
curl -I http://localhost:8080
# 期待されるヘッダ: HTTP/1.1 200 OK
```

## よくある問題と対処

- 403 Forbidden（原因）
  - nginx のエラーログに `directory index of "/var/www/html/" is forbidden` と表示される場合は、ドキュメントルートに `index.php` か `index.html` が無いために発生します。
  - 対処: プロジェクトルートに `index.php` または `index.html` を追加するか、nginx の設定でフォールバックを変更します。今回、確認用に `index.php` を追加して 200 を確認しています。

- パーミッション問題
  - マウントされたホストファイルが nginx/コンテナから読み取れない場合、権限を確認してください。通常は所有者が `root` でも nginx は読み取れるはずですが、特殊な環境では rootless Docker や macOS のファイル共有設定が影響する場合があります。

## PHP バージョンの変更について
- 現在の `docker-compose.yml` では `php` サービスのイメージを `php:8.4-fpm` にしています（Drupal 11.2 対応のため）。
- 将来的に追加の PHP 拡張（例: `pdo_pgsql`, `mbstring`, `xml`, `gd`, `zip` など）が必要な場合は、公式イメージに対して `Dockerfile` を作成し拡張をインストールすることを推奨します。例:

```Dockerfile
FROM php:8.4-fpm
RUN apt-get update && apt-get install -y \
    libzip-dev libpng-dev libxml2-dev zlib1g-dev \
  && docker-php-ext-install pdo pdo_pgsql mbstring xml zip gd
```

その `Dockerfile` をビルドするように `docker-compose.yml` を書き換えます（`build: ./.docker/php` など）。

## 追加の推奨
- Drupal 導入のための composer の追加・セットアップ。
- 開発用の .env 設定ファイルの用意（DB 接続情報など）と `settings.php` のテンプレート。
- `php` コンテナで必要な拡張を Dockerfile でインストールしておく。
- nginx の設定をもう少し堅牢に（404 フォールバック、セキュリティヘッダ、キャッシュ設定など）。

---

作成者: 実装スクリプト（自動）
最終更新: 2025-11-06
