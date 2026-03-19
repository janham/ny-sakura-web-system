# NAT Gateway 自動化 - Quick Start

## 概要

バッチサーバーにインターネットアクセスを提供するためのAnsible自動化。

## ファイル構成

```
ansible/
├── playbooks/
│   ├── enable_batch_internet.yml   # NAT Gateway設定用Playbook（新規）
│   ├── web.yml                      # gateway_nat ロール追加（更新）
│   └── batch.yml                    # gateway_client ロール追加（更新）
│
└── roles/
    ├── gateway_nat/                 # Webサーバー NAT設定ロール（新規）
    │   ├── defaults/main.yml       # デフォルト変数
    │   ├── tasks/main.yml          # タスク定義
    │   ├── handlers/main.yml       # ハンドラー定義
    │   └── README.md               # ロール説明
    │
    └── gateway_client/              # バッチサーバー Gateway設定ロール（新規）
        ├── defaults/main.yml       # デフォルト変数
        ├── tasks/main.yml          # タスク定義
        └── README.md               # ロール説明
```

## クイックスタート

### インターネットアクセスを有効化

```bash
# 1. 構文チェック
ansible-playbook playbooks/enable_batch_internet.yml --syntax-check

# 2. ドライラン（推奨）
ansible-playbook playbooks/enable_batch_internet.yml --check --diff

# 3. 実行
ansible-playbook playbooks/enable_batch_internet.yml -v
```

### 接続テスト

```bash
# バッチサーバーからインターネット接続確認
ssh gsuser@gs-batch -p 22 'ping -c 3 8.8.8.8'
ssh gsuser@gs-batch -p 22 'curl -I https://www.google.com'
```

## 設定のカスタマイズ

### ネットワークインターフェース変更

`roles/gateway_nat/defaults/main.yml`:
```yaml
gateway_public_interface: eth0   # パブリックインターフェース
gateway_private_interface: eth1  # プライベートインターフェース
```

### デバッグログ有効化

`roles/gateway_nat/defaults/main.yml`:
```yaml
gateway_enable_logging: true     # iptablesログ有効化
gateway_log_prefix: "NAT-DBG: "  # ログプレフィックス
```

### 接続テストをスキップ

`roles/gateway_client/defaults/main.yml`:
```yaml
gateway_verify_connectivity: false  # インターネット接続テストをスキップ
```

## 詳細ドキュメント

- **自動化ガイド:** `/docs/network/Ansible_NAT_Gateway_Automation.md`
- **手動設定手順:** `/docs/network/Batch_Server_Internet_Access_Setup.md`
- **gateway_nat ロール:** `roles/gateway_nat/README.md`
- **gateway_client ロール:** `roles/gateway_client/README.md`

## 注意事項

- 既存のネットワーク設定に影響する可能性があります
- 必ずドライラン（`--check --diff`）で事前確認してください
- 本番環境での実行前にテスト環境での検証を推奨します
- NAT設定後は、バッチサーバーのデフォルトゲートウェイが変更されます

## トラブルシューティング

### インターネットに接続できない

1. ゲートウェイ設定確認:
   ```bash
   ssh gsuser@gs-batch -p 22 'ip route show default'
   ```

2. IPフォワーディング確認:
   ```bash
   ssh gsuser@gs-web -p 26622 'cat /proc/sys/net/ipv4/ip_forward'
   ```

3. iptablesルール確認:
   ```bash
   ssh gsuser@gs-web -p 26622 'sudo iptables -t nat -L POSTROUTING -n -v'
   ```

詳細は `/docs/network/Ansible_NAT_Gateway_Automation.md` のトラブルシューティングセクションを参照。
