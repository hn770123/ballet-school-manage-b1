# マルチテナントアーキテクチャ

**作成日**: 2024年12月
**更新日**: 2025年12月

---

## 2.1 設計方針

**シングルバイナリ・マルチテナント**

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare Workers                       │
│                   (Single Binary App)                       │
├─────────────────────────────────────────────────────────────┤
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │ Tenant A  │  │ Tenant B  │  │ Tenant C  │  ...          │
│  │(バレエ教室)│  │(ダンス教室)│  │ (音楽教室) │               │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘               │
│        │              │              │                     │
│        └──────────────┼──────────────┘                     │
│                       ↓                                     │
│              ┌─────────────────┐                           │
│              │   Cloudflare D1  │                           │
│              │  (Shared DB with │                           │
│              │   tenant_id)     │                           │
│              └─────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

**選択理由**:
- **運用コスト最小化**: 1つのインスタンスで複数教室を運営
- **デプロイの簡素化**: バージョンアップが全テナントに即時反映
- **リソース効率**: Cloudflare Workers の無料枠を最大活用
- **横展開容易性**: 新規テナント追加はデータ登録のみ

## 2.2 テナント識別

#### URLルーティング方式

| 方式 | 例 | 採用 |
|------|-----|------|
| サブドメイン | `ballet.example.com` | ○ 推奨 |
| パスプレフィックス | `example.com/ballet/` | △ 代替 |
| クエリパラメータ | `example.com?tenant=ballet` | × 非推奨 |

**サブドメイン方式のメリット**:
- LINE公式アカウントとの1対1マッピングが明確
- LIFF URLの分離が容易
- テナント間の誤アクセス防止

#### テナント解決フロー

```
[リクエスト受信]
       ↓
[Host ヘッダーからサブドメイン抽出]
       ↓
[tenants テーブルで tenant_id を解決]
       ↓
[リクエストコンテキストに tenant_id を設定]
       ↓
[以降の全クエリに tenant_id フィルタを適用]
```

## 2.3 データ分離戦略

**論理分離（Shared Database, Shared Schema）**

- 全テナントが同一スキーマを共有
- 全テーブルに `tenant_id` カラムを追加
- クエリレベルでのテナント分離を強制

```sql
-- 例: 生徒テーブル
CREATE TABLE students (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,  -- テナント識別子
  name TEXT NOT NULL,
  studio TEXT NOT NULL,
  ...
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- 全クエリに tenant_id 条件を付与
SELECT * FROM students WHERE tenant_id = :current_tenant_id AND ...
```

**分離の強制**:
- アプリケーション層で全クエリにテナントフィルタを自動適用
- Row Level Security (RLS) 相当の実装をミドルウェアで実現
- 監査ログでテナント越境アクセスを検知

## 2.4 テナント管理

#### テナントテーブル

```sql
CREATE TABLE tenants (
  id TEXT PRIMARY KEY,              -- 'ballet-kids-jp'
  name TEXT NOT NULL,               -- '子供バレエ教室'
  subdomain TEXT UNIQUE NOT NULL,   -- 'ballet'
  status TEXT NOT NULL DEFAULT 'active', -- 'active', 'suspended', 'trial'
  plan TEXT NOT NULL DEFAULT 'free',     -- 'free', 'basic', 'premium'
  settings JSON NOT NULL,           -- テナント固有設定
  line_channel_id TEXT,             -- LINE公式アカウントのチャネルID
  line_channel_secret TEXT,         -- チャネルシークレット（暗号化）
  liff_id TEXT,                     -- LIFF ID
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

#### テナント設定（settings JSON）

```json
{
  "default_language": "ja",
  "supported_languages": ["ja", "en", "ru"],
  "timezone": "Asia/Tokyo",
  "features": {
    "recital_management": true,
    "lesson_management": true,
    "multilingual": true,
    "line_integration": true
  },
  "branding": {
    "primary_color": "#E91E63",
    "logo_url": "https://..."
  },
  "limits": {
    "max_students": 100,
    "max_admins": 5
  }
}
```

## 2.5 LINE連携とマルチテナント

各テナントは独自のLINE公式アカウントを持つ。

```
┌─────────────────┐     ┌─────────────────┐
│ Tenant A        │     │ Tenant B        │
│ LINE公式アカウント│     │ LINE公式アカウント│
│ (Channel ID: X) │     │ (Channel ID: Y) │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ↓
         ┌─────────────────────┐
         │  Webhook Endpoint   │
         │  /api/line/webhook  │
         └──────────┬──────────┘
                    ↓
         [Channel ID から tenant_id を解決]
                    ↓
         [テナントコンテキストで処理]
```

**Webhook での テナント識別**:
1. LINE からの Webhook リクエストに含まれる署名を検証
2. `destination` フィールドから Channel ID を取得
3. `tenants` テーブルで Channel ID → tenant_id を解決
4. 以降の処理はテナントコンテキスト内で実行
