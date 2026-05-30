# GS新サーバー構築プロジェクト

## プロジェクト概要

GS系Webアプリケーション用のさくらのクラウドサーバー構築・運用を管理するAnsibleプロジェクト。

- **ドキュメント**: `/Users/hironori/Obsidian/20_project/gs-portal/docs/`
- **Ansibleディレクトリ**: `./ansible/`

---

## サーバー構成

| ホスト名 | 接続先 | 役割 |
|---------|--------|------|
| gs-web | 133.125.90.129:26622 | Webサーバー（nginx + PHP） |
| gs-batch | 192.168.0.20:22（gs-web 経由 ProxyJump） | バッチサーバー |
| DBサーバー | MariaDB アプライアンス（vault管理） | DB（冗長化） |
| ファイルサーバー | NFS（vault管理） | 共有ストレージ /mnt/shared |

- **SSHユーザー**: gsuser
- **SSH鍵**: `~/.ssh/202602_sakura_key`
- **Vaultパスワード**: `~/.ansible_vault_pass`
- **OS**: Rocky Linux 9.7

---

## アプリケーション構成

### VirtualHost（3サイト）

| サイト | ドメイン | PHP | ドキュメントルート |
|--------|---------|-----|-----------------|
| supplier2 | supplier2.mobileselection.jp | PHP 8.1 | /var/www/app/supplier2/public |
| portal2 | portal2.mobileselection.jp | PHP 7.4 (Remi) | /var/www/app/portal2/public |
| portal2-new | portal2-new.mobileselection.jp | PHP 8.1 | /var/www/app/portal2-new/public |

### PHP-FPM ソケット

| バージョン | ソケットパス |
|-----------|------------|
| PHP 8.1 | `/run/php-fpm/www.sock` |
| PHP 7.4 | `/var/opt/remi/php74/run/php-fpm/www.sock` |

---

## Ansible 実行コマンド

作業ディレクトリは `./ansible/` で実行すること。

```bash
cd ansible

# Webサーバー全体構築
ansible-playbook playbooks/web.yml

# バッチサーバー全体構築
ansible-playbook playbooks/batch.yml

# タグ指定（部分実行）
ansible-playbook playbooks/web.yml --tags nginx_geo
ansible-playbook playbooks/web.yml --tags ssl
ansible-playbook playbooks/web.yml --tags nginx

# ドライラン（差分確認）
ansible-playbook playbooks/web.yml --tags nginx_geo --check --diff
```

---

## Geo Blocking（nginx geo module）

### 概要

- APNIC から日本IPv4レンジを自動取得（約4685件）
- 海外IPは HTTP 444（接続即切断）でブロック
- `geo_jp_only.conf` の `default no;` が正常な状態（`yes` になっている場合は Geo Blocking 無効＝異常）

### 許可対象

- 日本IPv4レンジ（APNIC）
- プライベートIP: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- Localhost: 127.0.0.0/8, ::1/128
- 個別許可: 35.213.74.82

### サイト別アクセス制御

| サイト | パス | Geo Blocking |
|--------|------|-------------|
| supplier2 | 全パス | 有効（日本のみ） |
| portal2 | 全パス | 有効（日本のみ） |
| portal2-new | `/api/smaregi/webhooks/*` | **無効**（全世界許可） |
| portal2-new | それ以外 | 有効（日本のみ） |

### webhook location の注意事項

`/api/smaregi/webhooks/` は `try_files` を使わず直接 PHP-FPM に渡す実装になっている。

`try_files` を使うと `/index.php` への内部リダイレクトが発生し、geo blocking 付きの `location ~ \.php$` に再マッチして海外IPがブロックされるため。

```nginx
location ~ ^/api/smaregi/webhooks/ {
    include fastcgi_params;
    fastcgi_pass unix:/run/php-fpm/www.sock;
    fastcgi_param SCRIPT_FILENAME $document_root/index.php;
    fastcgi_param SCRIPT_NAME     /index.php;
}
```

### 日本IPレンジの更新（月次推奨）

```bash
cd ansible
ansible-playbook playbooks/web.yml --tags nginx_geo
```

---

## SSL証明書

- **方式**: Let's Encrypt（certbot）
- **自動更新**: 設定済み
- **対象ドメイン**: supplier2, portal2, portal2-new（各サブドメイン）

---

## rsync 同期（Web → Batch）

- Webサーバーをマスターとしてバッチサーバーへ5分間隔で同期
- 同期対象: portal2-new のソースコードのみ
- **除外**: `.env`, `storage/`, `cache/`（各サーバーで独立管理）

---

## NAT Gateway

バッチサーバーはプライベートネットワークのみのため、Webサーバー（192.168.0.10）を NAT Gateway として使用してインターネットアクセスする。

---

## ロール構成

```
roles/
├── common/          # OS基本設定（タイムゾーン、ロケール、パッケージ等）
├── nginx/           # nginx インストール・VirtualHost設定
├── nginx_geo/       # Geo Blocking（nginx geo module）
├── php81/           # PHP 8.1 インストール・設定
├── php74/           # PHP 7.4 (Remi) インストール・設定
├── ssl/             # Let's Encrypt SSL証明書
├── mariadb_client/  # MariaDB クライアント設定
├── nfs/             # NFS マウント設定
├── composer/        # Composer インストール
├── laravel_app/     # Laravelアプリケーション設定
├── laravel_scheduler/   # Laravelスケジューラー（cron）
├── laravel_logrotate/   # ログローテーション
├── supervisor/      # Supervisord（キューワーカー管理）
├── rsync_to_batch/  # WebからBatchへのrsync同期
├── gateway_nat/     # NAT Gateway設定（Webサーバー側）
└── gateway_client/  # NAT Gateway クライアント設定（バッチサーバー側）
```
