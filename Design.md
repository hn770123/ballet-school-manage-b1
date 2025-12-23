# 教室運営支援システム 基本設計書

**作成日**: 2024年12月
**更新日**: 2025年12月
**バージョン**: 3.0

-----

## 目次

1. [システム概要](#1-システム概要)
2. [マルチテナントアーキテクチャ](#2-マルチテナントアーキテクチャ)
3. [設計思想](#3-設計思想)
4. [システム構成](#4-システム構成)
5. [認証設計](#5-認証設計)
6. [データモデル設計](#6-データモデル設計)
7. [セキュリティ設計](#7-セキュリティ設計)
8. [多言語対応戦略](#8-多言語対応戦略)
9. [画面構成](#9-画面構成)
10. [通知設計](#10-通知設計)
11. [開発計画](#11-開発計画)
12. [技術スタック](#12-技術スタック)
13. [運用コスト](#13-運用コスト)
14. [設計課題](#14-設計課題)
15. [期待される効果](#15-期待される効果)

-----

## 1. システム概要

習い事教室における**日々のレッスン運営**と**発表会・イベント運営**の課題を解決する多言語対応マルチテナントWebシステム。

### 1.0 マルチテナント対応

本システムは**シングルバイナリ・マルチテナント**設計を採用し、1つのアプリケーションで複数の教室（テナント）を運営可能。

**対応可能な教室種別**:
- バレエ教室
- ダンススクール
- 音楽教室
- 体操教室
- その他習い事教室

**テナント固有設定**: 各テナントの固有設定は別ファイル（例: `TenantConfig-BalletSchool.md`）で管理。

### 1.1 コアバリュー

- **情報の可視化**: 「何が決まっていて何が決まっていないか」を明確化
- **芸術的判断の尊重**: 直前までの変更に対応できる柔軟性
- **言語の壁の解消**: 日本語・英語・ロシア語での情報アクセス
- **双方向コミュニケーション**: 保護者からのステータス報告・問い合わせの構造化
- **プライバシー保護**: 個人の行動情報（いつ・誰が・どこにいるか）を適切に保護

### 1.2 主要機能

#### レッスン管理
- 通常レッスンスケジュールの管理
- **臨時レッスン**の通知と参加/不参加の応答
- **休校連絡**と確認応答の管理
- 欠席連絡の受付

#### 発表会管理
- 発表会・演目・キャスティング管理
- 準備物（衣装・小道具）のステータス追跡
- 裏方タスクの管理と割り当て

#### 共通機能
- 多言語対応（選択的翻訳方式）
- LINE通知連携
- **プライバシーに配慮した情報公開制御**

-----

## 2. マルチテナントアーキテクチャ

### 2.1 設計方針

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

### 2.2 テナント識別

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

### 2.3 データ分離戦略

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

### 2.4 テナント管理

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

### 2.5 LINE連携とマルチテナント

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

-----

## 3. 設計思想

### 3.1 普遍的なプロジェクト構造

本システムは「バレエ教室の発表会管理」という具体的な課題を解決しますが、その本質は以下の普遍的なプロジェクト構造に基づいています：

**プロジェクト管理の基本構造**

```
[企画・計画]
    ↓
[マイルストーン設定] → チェックポイントの定義
    ↓
[タスク分解] → 依存関係の明確化
    ↓
[担当者割り当て] → 責任の明確化
    ↓
[メンバーへの周知] → 情報共有と通知
    ↓
[進捗追跡] → チェックリストとステータス管理
    ↓
[完了確認] → Definition of Done
```

この構造は、ソフトウェア開発、イベント企画、製品発売準備など、あらゆるプロジェクトに共通します。

### 3.2 柔軟性の確保

**固定化を避ける設計**

- 衣装・小道具・裏方タスクを統一的な「準備物・タスク」概念として扱う
- カテゴリやステータスは管理者が定義可能
- 関連付けは多対多で柔軟に設定

**変更への対応**

- 演目・出演者の直前変更に対応
- 準備物の追加・削除が容易
- ステータスのカスタマイズが可能

**テンプレートによる効率化**

- 過去のプロジェクト（発表会）をテンプレートとして再利用
- 定型タスクのパターン化
- ベストプラクティスの蓄積

### 3.3 情報の可視化

**決定事項と未定事項の分離**

- 確定フラグによる明示的な状態管理
- 「TBD（未定）」状態の可視化
- 変更履歴の追跡（いつ、誰が、何を変更したか）

**ステータスの一元管理**

- 準備物の進捗を個別に追跡
- 未対応事項の自動検出
- ダッシュボードでの全体把握

**多角的なビュー**

- リスト表示：詳細な情報確認
- カンバン表示：ステータス別の俯瞰
- タイムライン表示：期日ベースの計画確認
- 依存関係図：タスク間の関係性の可視化

### 3.4 責任の明確化

**RACI モデルの簡略適用**

- **Responsible（実行者）**: 誰がタスクを実行するか
- **Accountable（責任者）**: 誰が最終確認するか
- **Informed（通知先）**: 誰に進捗を伝えるか

**完了条件（Definition of Done）の明示**

- 何をもって「完了」とするかの定義
- 曖昧な状態の排除
- 認識のずれの防止

### 3.5 依存関係とクリティカルパス

**タスク間の依存関係**

- 「AタスクはBタスクの完了後に開始」という関係性の明示
- ブロッカーの可視化（何が進行を妨げているか）
- 依存関係に基づく自動アラート

**マイルストーンによる進捗管理**

- 中間チェックポイントの設定
- マイルストーンごとの達成状況の可視化
- 遅延の早期発見

### 3.6 運用負荷の最小化

**最小限の運用体制**

- 先生 + システム + 片手間の補助で運営可能
- 自動翻訳による多言語対応
- LINE通知による最低限の連絡

**効率的な翻訳戦略**

- 編集時に翻訳を事前生成（リアルタイム翻訳を避ける）
- 翻訳対象の選別（必要なものだけ翻訳）
- コンテキスト情報の活用

-----

## 4. システム構成

### 4.1 技術基盤

**Cloudflare エッジコンピューティング + LINE Platform**

|コンポーネント           |役割                    |
|------------------|----------------------|
|Cloudflare Workers|API・Webアプリケーションのホスティング|
|Cloudflare D1     |SQLiteベースのデータベース      |
|Workers AI        |多言語翻訳、コンテキスト解釈        |
|LINE公式アカウント|保護者との主要コミュニケーションチャネル|
|LIFF (LINE Front-end Framework)|LINE内で動作するWebアプリケーション|
|LINE Messaging API|メッセージ送受信（Reply中心）|
|LINE Login|保護者の認証・識別|

### 4.2 LINE Platform 構成

#### LINE公式アカウント

保護者との主要な接点となるLINE公式アカウントを開設。

**役割**:
- 保護者がLINEアプリ内から直接システムにアクセス
- リッチメニューによるナビゲーション
- Webhookによるメッセージ受信と自動応答

**機能**:
- 友だち追加時の初期登録案内
- キーワード応答による情報取得
- LIFFアプリへの導線

#### LIFF（LINE Front-end Framework）

LINE内で動作するWebアプリケーションとして実装。

**メリット**:
- LINEアプリ内でシームレスに動作
- LINE Loginによる自動認証（ユーザー識別）
- ネイティブアプリのインストール不要
- プッシュ通知なしでもユーザーがプル型で情報取得可能

**LIFFアプリ構成**:

| アプリ名 | 画面サイズ | 用途 |
|---------|-----------|------|
| メイン | Full | 児童情報登録、準備物一覧、ステータス更新 |
| タスク詳細 | Tall | 個別タスクの詳細表示・更新 |
| クイック確認 | Compact | ステータスのクイック確認 |

### 4.3 アーキテクチャ

```
[保護者]
    │
    ├─[LINE公式アカウント]─────────────────────────┐
    │       │                                      │
    │       ├─[リッチメニュー]→[LIFFアプリ]        │
    │       │                     │                │
    │       ├─[メッセージ送信]→[Webhook]           │
    │       │                     │                │
    │       └─[キーワード応答]←[Reply Token]       │
    │                             │                │
    └─[Webブラウザ（管理者）]     │                │
              │                   │                │
              ↓                   ↓                │
        [Cloudflare Workers]←────┘                │
              │                                    │
    ┌─────────┼─────────┬─────────┐               │
    │         │         │         │               │
    ↓         ↓         ↓         ↓               │
┌───────┐┌───────┐┌─────────┐┌──────────┐        │
│ D1 DB ││Workers││  LINE   ││   LINE   │        │
│(データ)││  AI   ││Messaging││  Login   │←───────┘
│       ││(翻訳) ││   API   ││  (認証)  │
└───────┘└───────┘└─────────┘└──────────┘
```

### 4.4 データフロー

#### 保護者の操作フロー（プル型）

**最初の保護者（新規登録）**:

1. 保護者（例: 母）がLINE公式アカウントを友だち追加
2. リッチメニューからLIFFアプリを起動
3. LINE Loginで自動認証（LINE User ID取得）
4. 初回登録画面で「新規登録」を選択
5. 児童情報を登録（名前、教室、自分との関係）
6. システムがアクセスコードを発行（例: `HNK-7842`）
7. `line_accounts` に LINEアカウント情報を保存
8. `students` に生徒情報を保存
9. `line_student_access` に紐づけを保存（is_primary=true）
10. 準備物・タスクの一覧を閲覧
11. ステータスを更新・コメントを投稿

**追加の保護者（アクセスコード入力）**:

1. 保護者（例: 父）がLINE公式アカウントを友だち追加
2. リッチメニューからLIFFアプリを起動
3. LINE Loginで自動認証（LINE User ID取得）
4. 初回登録画面で「アクセスコード入力」を選択
5. 母から共有されたアクセスコードを入力
6. 自分と児童の関係を選択（父）
7. `line_accounts` に LINEアカウント情報を保存
8. `line_student_access` に紐づけを保存（is_primary=false）
9. 同じ生徒の情報にアクセス可能に

```
[データの流れ]
父のLINE ──┐
           ├── line_student_access ──→ 生徒（花子）
母のLINE ──┘                              ↓
                                    準備物・タスク
```

#### メッセージ応答フロー（Reply Token）

1. 保護者がLINE公式アカウントにメッセージ送信
1. Webhook経由でCloudflare Workersが受信
1. メッセージ内容を解析（キーワードマッチング）
1. Reply Tokenを使って即座に応答（無料）
1. 必要に応じてLIFFアプリへの導線を案内

#### 管理者の操作フロー

1. 管理者がWebブラウザから管理画面にアクセス
1. 情報を入力/更新
1. 必要に応じて翻訳を生成（Workers AI）
1. 翻訳結果をDBに保存
1. **緊急時のみ**手動でプッシュ通知を送信

-----

## 5. 認証設計

### 5.1 LINE認証（保護者向け）

LINE公式アカウント経由でLIFFアプリにアクセスする保護者は、LINE Loginにより自動的に認証される。

**特徴**

- LINE公式アカウントを友だち追加するだけで利用開始
- LIFFアプリ起動時にLINE User IDを自動取得
- パスキーの発行・管理が不要
- **複数の保護者（父・母・祖父母など）が同じ子供にアクセス可能**
- **アクセスコードによる柔軟な紐づけ**

**運用フロー（初回登録：最初の保護者）**

1. 管理者がLINE公式アカウントの友だち追加用QRコードを配布
2. 保護者（例: 母）がQRコードからLINE公式アカウントを友だち追加
3. 友だち追加時のウェルカムメッセージでLIFFアプリへの導線を案内
4. 保護者がLIFFアプリを起動、LINE Loginで自動認証
5. 初回アクセス時に児童情報を登録（児童名、教室、自分との関係）
6. システムが生徒ごとに**アクセスコード**を自動発行（例: `HNK-7842`）
7. LINE User IDと生徒が `line_student_access` テーブルで紐づけ

**運用フロー（追加登録：他の保護者）**

1. 最初の保護者が他の保護者（例: 父）にアクセスコードを共有
2. 父がLINE公式アカウントを友だち追加、LIFFアプリを起動
3. 「子供を追加」画面でアクセスコードを入力
4. 父のLINEアカウントが同じ生徒に紐づけられる（関係: 父）
5. 必要に応じて祖父母なども同様に追加可能

```
[紐づけの例]
母のLINE (is_primary=true) ─┬─ 長女（花子）access_code: HNK-7842
父のLINE                   ─┤
祖母のLINE                 ─┘

母のLINE (is_primary=true) ─┬─ 次女（桜子）access_code: SKR-3156
父のLINE                   ─┘
```

**スコープ**

- LINEアカウント単位で認証（1人の大人 = 1つのLINEアカウント）
- 1つのLINEアカウントで複数の生徒にアクセス可能（兄弟姉妹）
- 複数のLINEアカウントが同じ生徒にアクセス可能（父・母・祖父母）
- `is_primary` フラグで主担当を設定（通知の優先送信先）

**LINE未使用者への対応**

LINEを使用しない保護者向けに、従来のパスキー認証も併用可能。

- 形式: `ballet-sakura-7842` のような覚えやすい文字列
- Webブラウザから直接アクセス
- 管理者が個別に発行・配布
- 生徒との紐づけにはアクセスコードを使用

### 5.2 パスワード認証（管理者向け）

管理者はWebブラウザから従来のパスワード認証でアクセス。

**役割と権限**

|役割       |権限                             |
|---------|-------------------------------|
|先生（owner）|全データの閲覧・編集、演目・出演者の確定、手動プッシュ通知送信|
|サポート保護者  |情報の補足入力、問い合わせ対応、パスキー発行（LINE未使用者向け）|
|一般保護者    |自分の子供の情報閲覧、準備ステータスの更新、コメント投稿   |

-----

## 6. データモデル設計

### 6.1 設計原則

**柔軟性を重視**

- 固定的なエンティティを避ける
- カテゴリ・ステータスは管理者が定義
- 属性は JSON 形式で拡張可能
- 多対多の関連付けで柔軟性を確保

### 6.2 主要エンティティ

#### 6.2.1 基本情報

**LINEアカウント (Line Accounts)**

LINEユーザー単位の認証情報を管理。1人の大人が1つのLINEアカウントを持つ。
※ LINEアカウントはテナントをまたいで共有可能（同一ユーザーが複数教室を利用するケース）

```sql
CREATE TABLE line_accounts (
  id TEXT PRIMARY KEY,
  line_user_id TEXT NOT NULL, -- LINE User ID（LIFF経由でログイン時に取得）
  line_display_name TEXT, -- LINEの表示名
  preferred_language TEXT DEFAULT 'ja', -- 'ja', 'en', 'ru'
  passkey TEXT, -- LINE未使用者向け（併用可能）
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  UNIQUE(line_user_id) -- LINE User ID はグローバルで一意
);
```

**LINEアカウントとテナントの紐づけ (Line Account Tenants)**

LINEアカウントとテナントの多対多関係を管理。

```sql
CREATE TABLE line_account_tenants (
  id TEXT PRIMARY KEY,
  line_account_id TEXT NOT NULL,
  tenant_id TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'guardian', -- 'owner', 'admin', 'guardian'
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id),
  FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  UNIQUE(line_account_id, tenant_id)
);
```

**生徒 (Students)**

生徒情報は保護者から独立したエンティティとして管理。複数のLINEアカウントから参照可能。

```sql
CREATE TABLE students (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  name TEXT NOT NULL,
  studio TEXT NOT NULL, -- テナント設定で定義された教室ID
  access_code TEXT NOT NULL, -- 他の保護者が紐づけるためのコード（例: 'ABC-1234'）
  access_code_expires_at TEXT, -- アクセスコードの有効期限（null=無期限）
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  UNIQUE(tenant_id, access_code) -- テナント内でアクセスコードは一意
);
```

**LINEアカウントと生徒の紐づけ (Line Student Access)**

LINEアカウントと生徒の多対多関係を管理。複数の保護者（父・母・祖父母など）が同じ生徒にアクセス可能。

```sql
CREATE TABLE line_student_access (
  id TEXT PRIMARY KEY,
  line_account_id TEXT NOT NULL,
  student_id TEXT NOT NULL,
  relation_type TEXT NOT NULL, -- 'mother', 'father', 'grandparent', 'other'
  relation_label TEXT, -- 関係の表示名（例: '祖母（母方）'）
  is_primary BOOLEAN DEFAULT FALSE, -- 主担当かどうか（通知の優先送信先）
  can_edit BOOLEAN DEFAULT TRUE, -- 編集権限の有無
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id),
  FOREIGN KEY (student_id) REFERENCES students(id),
  UNIQUE(line_account_id, student_id)
);
```

**設計のポイント: LINEアカウントと生徒のn対n関係**

従来の設計では「1つのLINEアカウント = 1人の保護者 = その保護者の子供たち」という1対多の構造でした。
しかし実際には：

- **複数の大人が同じ子供にアクセス**: 父・母・祖父母など複数の保護者が同じ子供の情報を確認したい
- **1人の大人が複数の子供を管理**: 兄弟姉妹がいる場合
- **例**: 父・母・祖母の3人 × 子供3人 = 最大9つの紐づけ

この問題を解決するため、中間テーブル `line_student_access` を導入し、柔軟な多対多関係を実現。

```
[LINEアカウント]              [生徒]
 ├─ 父のLINE ─────┬─────── 長女（花子）
 ├─ 母のLINE ─────┼─────── 次女（桜子）
 └─ 祖母のLINE ───┴─────── 長男（太郎）
```

**アクセスコードによる紐づけフロー**:

1. 最初の保護者（例: 母）がLIFFで子供を登録
2. システムが生徒ごとに `access_code` を自動発行（例: `HNK-7842`）
3. 母が父に `access_code` を共有（LINEで送信など）
4. 父がLIFFで「子供を追加」→ アクセスコードを入力
5. 父のLINEアカウントも同じ生徒に紐づけられる

#### 6.2.2 レッスン管理

**レッスンスケジュール (Lesson Schedules)**

通常の定期レッスンスケジュールを管理。

```sql
CREATE TABLE lesson_schedules (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  studio TEXT NOT NULL, -- テナント設定で定義された教室ID
  day_of_week INTEGER NOT NULL, -- 0=日曜, 1=月曜, ..., 6=土曜
  start_time TEXT NOT NULL, -- 'HH:MM' 形式
  end_time TEXT NOT NULL, -- 'HH:MM' 形式
  class_name TEXT, -- クラス名（例: '幼児クラス', '小学生クラス'）
  is_active BOOLEAN DEFAULT TRUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);
```

**レッスンイベント (Lesson Events)**

通常レッスン・臨時レッスン・休校を管理。通常レッスンは定期スケジュールから自動生成も可能。

```sql
CREATE TABLE lesson_events (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  schedule_id TEXT, -- 定期スケジュールから生成された場合に紐づけ（臨時レッスンはnull）
  studio TEXT NOT NULL, -- テナント設定で定義された教室ID
  event_type TEXT NOT NULL, -- 'regular'（通常）, 'extra'（臨時）, 'cancelled'（休校）
  event_date TEXT NOT NULL, -- ISO 8601（YYYY-MM-DD）
  start_time TEXT NOT NULL, -- 'HH:MM' 形式
  end_time TEXT NOT NULL, -- 'HH:MM' 形式
  class_name TEXT, -- クラス名
  title TEXT, -- イベントのタイトル（臨時レッスン・休校の場合に使用）
  description TEXT, -- 詳細説明
  target_type TEXT NOT NULL, -- 'all_studio'（教室全体）, 'class'（特定クラス）, 'selected'（選択された生徒）
  response_deadline TEXT, -- 応答締切（ISO 8601）
  created_by TEXT NOT NULL, -- 作成した管理者ID
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  FOREIGN KEY (schedule_id) REFERENCES lesson_schedules(id)
);
```

**レッスン対象生徒 (Lesson Event Targets)**

臨時レッスンや休校の対象となる生徒を管理。`target_type='selected'` の場合に使用。

```sql
CREATE TABLE lesson_event_targets (
  id TEXT PRIMARY KEY,
  lesson_event_id TEXT NOT NULL,
  student_id TEXT NOT NULL,
  created_at TEXT NOT NULL,
  FOREIGN KEY (lesson_event_id) REFERENCES lesson_events(id),
  FOREIGN KEY (student_id) REFERENCES students(id),
  UNIQUE(lesson_event_id, student_id)
);
```

**レッスン応答 (Lesson Responses)**

臨時レッスンへの参加/不参加、休校連絡の確認応答を管理。

```sql
CREATE TABLE lesson_responses (
  id TEXT PRIMARY KEY,
  lesson_event_id TEXT NOT NULL,
  student_id TEXT NOT NULL,
  response_type TEXT NOT NULL, -- 'attend'（参加）, 'absent'（不参加/欠席）, 'acknowledged'（確認済み）
  responded_by TEXT NOT NULL, -- 応答した保護者のline_account_id
  response_note TEXT, -- 備考（欠席理由など）
  responded_at TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (lesson_event_id) REFERENCES lesson_events(id),
  FOREIGN KEY (student_id) REFERENCES students(id),
  FOREIGN KEY (responded_by) REFERENCES line_accounts(id),
  UNIQUE(lesson_event_id, student_id) -- 1生徒につき1応答
);
```

**設計のポイント: 臨時レッスンと休校のターゲット管理**

臨時レッスンは通常レッスンと異なり、特定の生徒グループのみを対象とする場合があります：

- **教室全体** (`target_type='all_studio'`): その教室の全生徒が対象
- **特定クラス** (`target_type='class'`): 指定されたクラスの生徒のみが対象
- **選択された生徒** (`target_type='selected'`): `lesson_event_targets` で指定された生徒のみが対象

```
[臨時レッスンの例]
イベント: 発表会前特別練習
対象: selected（選択された生徒）
  └── 対象生徒: 花子、桜子、太郎（発表会出演者のみ）

[休校の例]
イベント: 台風による休校
対象: all_studio（教室A全体）
  └── 教室Aの全生徒に通知
```

#### 6.2.3 発表会管理

**発表会 (Recitals)**

```sql
CREATE TABLE recitals (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  title TEXT NOT NULL,
  date TEXT, -- ISO 8601
  venue TEXT,
  status TEXT NOT NULL, -- 'planning', 'preparation', 'completed'
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);
```

**演目 (Programs)**

```sql
CREATE TABLE programs (
  id TEXT PRIMARY KEY,
  recital_id TEXT NOT NULL,
  title TEXT NOT NULL,
  order_num INTEGER,
  is_confirmed BOOLEAN DEFAULT FALSE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (recital_id) REFERENCES recitals(id)
);
```

**キャスティング (Casting)**

```sql
CREATE TABLE casting (
  id TEXT PRIMARY KEY,
  program_id TEXT NOT NULL,
  student_id TEXT NOT NULL,
  role_name TEXT, -- nullable: 役名がない場合もある
  is_confirmed BOOLEAN DEFAULT FALSE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (program_id) REFERENCES programs(id),
  FOREIGN KEY (student_id) REFERENCES students(id),
  UNIQUE(program_id, student_id, role_name)
);
```

#### 6.2.4 マイルストーン管理

**マイルストーン (Milestones)**

プロジェクトの中間チェックポイントを管理。

```sql
CREATE TABLE milestones (
  id TEXT PRIMARY KEY,
  recital_id TEXT NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  due_date TEXT, -- ISO 8601
  is_completed BOOLEAN DEFAULT FALSE,
  completed_at TEXT,
  order_num INTEGER,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (recital_id) REFERENCES recitals(id)
);
```

例:
- 「衣装サイズ確認完了」（本番2週間前）
- 「小道具準備完了」（本番1週間前）
- 「リハーサル」（本番3日前）

#### 6.2.5 準備物・タスク管理

**準備物・タスク定義 (Items)**

衣装・小道具・裏方タスクを統一的に管理する柔軟なモデル。

```sql
CREATE TABLE items (
  id TEXT PRIMARY KEY,
  recital_id TEXT NOT NULL,
  milestone_id TEXT, -- 関連するマイルストーン
  type TEXT NOT NULL, -- 'costume', 'prop', 'backstage_task', 'other'
  category TEXT, -- 自由なカテゴリ（例: 'tutu', 'lighting', 'transportation'）
  name TEXT NOT NULL,
  description TEXT,
  priority TEXT DEFAULT 'medium', -- 'critical', 'high', 'medium', 'low'
  due_date TEXT, -- ISO 8601
  done_criteria TEXT, -- 完了条件（Definition of Done）
  assignee_type TEXT, -- 'guardian', 'student', 'teacher', 'unassigned'
  assignee_id TEXT, -- 担当者ID（guardian_id, student_id, または null）
  reviewer_id TEXT, -- 確認者ID（guardian_id）
  attributes JSON, -- 柔軟な属性管理
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (recital_id) REFERENCES recitals(id),
  FOREIGN KEY (milestone_id) REFERENCES milestones(id)
);
```

**attributes の例**:

```json
{
  "acquisition_method": "purchase",
  "purchase_url": "https://...",
  "teacher_stock": 5,
  "making_instructions": "...",
  "quantity_needed": 3,
  "estimated_effort": "2 hours"
}
```

**タスク依存関係 (Item Dependencies)**

タスク間の依存関係を管理。

```sql
CREATE TABLE item_dependencies (
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL, -- 依存元（このタスクは...）
  depends_on_item_id TEXT NOT NULL, -- 依存先（...このタスクの完了後に開始）
  dependency_type TEXT DEFAULT 'finish_to_start', -- 'finish_to_start', 'start_to_start', 'finish_to_finish'
  created_at TEXT NOT NULL,
  FOREIGN KEY (item_id) REFERENCES items(id),
  FOREIGN KEY (depends_on_item_id) REFERENCES items(id)
);
```

例:
- 「衣装の縫製」は「生地の購入」の完了後に開始
- 「最終確認」は「サイズ調整」の完了後に開始

**ブロッカー (Blockers)**

タスクの進行を妨げる問題を追跡。

```sql
CREATE TABLE blockers (
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL,
  description TEXT NOT NULL,
  reported_by TEXT NOT NULL, -- line_account_id
  resolved_at TEXT,
  resolved_by TEXT, -- line_account_id
  resolution_notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (item_id) REFERENCES items(id),
  FOREIGN KEY (reported_by) REFERENCES line_accounts(id)
);
```

**関連付け (Item Relations)**

準備物・タスクと演目・役・生徒の柔軟な多対多関連。

```sql
CREATE TABLE item_relations (
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL,
  relation_type TEXT NOT NULL, -- 'program', 'role', 'student'
  relation_id TEXT NOT NULL,
  created_at TEXT NOT NULL,
  FOREIGN KEY (item_id) REFERENCES items(id)
);
```

例:

- `衣装A` → `演目X` に関連
- `衣装B` → `演目Y` の `役Z` に関連
- `裏方タスクC` → `生徒P` に割り当て

**ステータス追跡 (Item Status)**

```sql
CREATE TABLE item_status (
  id TEXT PRIMARY KEY,
  item_id TEXT NOT NULL,
  student_id TEXT, -- nullable: 生徒に紐づかないタスクもある
  status TEXT NOT NULL, -- 管理者が定義（例: 'not_started', 'in_progress', 'completed', 'issue'）
  acquisition_method TEXT, -- 個別の入手方法（購入/手作り/貸出/手持ち）
  notes TEXT,
  updated_by TEXT NOT NULL, -- line_account_id or 'system'
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (item_id) REFERENCES items(id),
  FOREIGN KEY (student_id) REFERENCES students(id)
);
```

**コメント・質問 (Comments)**

```sql
CREATE TABLE comments (
  id TEXT PRIMARY KEY,
  item_status_id TEXT NOT NULL,
  author_id TEXT NOT NULL, -- line_account_id
  content TEXT NOT NULL,
  is_question BOOLEAN DEFAULT FALSE,
  is_resolved BOOLEAN DEFAULT FALSE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (item_status_id) REFERENCES item_status(id),
  FOREIGN KEY (author_id) REFERENCES line_accounts(id)
);
```

#### 6.2.6 翻訳管理

**翻訳テーブル (Translations)**

編集時に事前生成した翻訳を保存。

```sql
CREATE TABLE translations (
  id TEXT PRIMARY KEY,
  source_table TEXT NOT NULL, -- 'programs', 'items', 'comments', etc.
  source_id TEXT NOT NULL,
  source_field TEXT NOT NULL, -- 'description', 'notes', etc.
  language TEXT NOT NULL, -- 'ja', 'en', 'ru'
  translated_text TEXT NOT NULL,
  tpo_context JSON, -- TPO情報
  created_at TEXT NOT NULL,
  UNIQUE(source_table, source_id, source_field, language)
);
```

**tpo_context の例**:

```json
{
  "timing": "urgent_notice",
  "context": "costume_preparation",
  "purpose": "request",
  "urgency": "urgent",
  "audience": "parents",
  "domain": "costume"
}
```

#### 6.2.7 バレエ用語辞書

**用語集 (Glossary)**

翻訳精度向上のためのコンテキスト提供。

```sql
CREATE TABLE glossary (
  id TEXT PRIMARY KEY,
  tenant_id TEXT, -- テナント識別子（null=システム共通用語）
  term TEXT NOT NULL,
  category TEXT, -- 'technique', 'costume', 'music', etc.
  translation_ja TEXT,
  translation_en TEXT,
  translation_ru TEXT,
  notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id),
  UNIQUE(tenant_id, term, category)
);
```

#### 6.2.8 監査ログ・変更履歴

**変更履歴 (Audit Log)**

すべての変更を追跡し、透明性と説明責任を確保。

```sql
CREATE TABLE audit_log (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  table_name TEXT NOT NULL, -- 変更されたテーブル名
  record_id TEXT NOT NULL, -- 変更されたレコードのID
  action TEXT NOT NULL, -- 'create', 'update', 'delete'
  changed_by TEXT NOT NULL, -- guardian_id or 'system'
  changed_by_type TEXT NOT NULL, -- 'guardian', 'admin', 'system'
  old_values JSON, -- 変更前の値（updateの場合）
  new_values JSON, -- 変更後の値
  change_summary TEXT, -- 人間が読める変更の要約
  created_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);
```

例:
```json
{
  "table_name": "items",
  "record_id": "item-123",
  "action": "update",
  "changed_by": "guardian-456",
  "old_values": {"status": "not_started", "priority": "medium"},
  "new_values": {"status": "in_progress", "priority": "high"},
  "change_summary": "ステータスを「未着手」から「進行中」に変更、優先度を「中」から「高」に変更"
}
```

#### 6.2.9 通知設定

**通知設定 (Notification Preferences)**

ユーザーごとの通知のカスタマイズ。

```sql
CREATE TABLE notification_preferences (
  id TEXT PRIMARY KEY,
  line_account_id TEXT NOT NULL,
  notification_type TEXT NOT NULL, -- 'item_update', 'deadline_reminder', 'comment_reply', 'milestone_update', etc.
  channel TEXT NOT NULL, -- 'line', 'email', 'in_app'
  is_enabled BOOLEAN DEFAULT TRUE,
  frequency TEXT DEFAULT 'immediate', -- 'immediate', 'daily_digest', 'weekly_digest'
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id),
  UNIQUE(line_account_id, notification_type, channel)
);
```

**通知履歴 (Notification Log)**

送信された通知の記録。

```sql
CREATE TABLE notification_log (
  id TEXT PRIMARY KEY,
  line_account_id TEXT NOT NULL,
  notification_type TEXT NOT NULL,
  channel TEXT NOT NULL,
  content TEXT NOT NULL,
  related_table TEXT, -- 関連するテーブル名
  related_id TEXT, -- 関連するレコードID
  sent_at TEXT NOT NULL,
  read_at TEXT, -- 既読時刻
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id)
);
```

#### 6.2.10 テンプレート機能

**テンプレート (Templates)**

過去のプロジェクトを再利用可能なテンプレートとして保存。

```sql
CREATE TABLE templates (
  id TEXT PRIMARY KEY,
  tenant_id TEXT, -- テナント識別子（null=システム共通テンプレート）
  name TEXT NOT NULL,
  description TEXT,
  source_recital_id TEXT, -- 元になった発表会（任意）
  category TEXT, -- 'recital', 'workshop', 'competition', etc.
  is_public BOOLEAN DEFAULT FALSE, -- 他のテナントでも使用可能か
  created_by TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);
```

**テンプレート項目 (Template Items)**

テンプレートに含まれるタスクのパターン。

```sql
CREATE TABLE template_items (
  id TEXT PRIMARY KEY,
  template_id TEXT NOT NULL,
  type TEXT NOT NULL,
  category TEXT,
  name TEXT NOT NULL,
  description TEXT,
  priority TEXT DEFAULT 'medium',
  days_before_event INTEGER, -- 本番の何日前が期日か
  done_criteria TEXT,
  dependencies JSON, -- 他のtemplate_itemへの依存関係
  attributes JSON,
  order_num INTEGER,
  created_at TEXT NOT NULL,
  FOREIGN KEY (template_id) REFERENCES templates(id)
);
```

テンプレート適用時:
1. 発表会の日付から各タスクの期日を自動計算
2. 依存関係を維持したまま新しいタスクを生成
3. 必要に応じてカスタマイズ可能

#### 6.2.11 LINE連携

**LINEメッセージ履歴 (LINE Message Log)**

LINE公式アカウント経由で送受信したメッセージの記録。

```sql
CREATE TABLE line_message_log (
  id TEXT PRIMARY KEY,
  line_account_id TEXT, -- 紐づけ済みのLINEアカウントID（未登録の場合はnull）
  line_user_id TEXT NOT NULL, -- LINE User ID
  direction TEXT NOT NULL, -- 'incoming'（受信）, 'outgoing'（送信）
  message_type TEXT NOT NULL, -- 'text', 'image', 'template', 'flex', etc.
  message_content TEXT, -- メッセージ内容（テキストの場合）
  reply_token TEXT, -- Reply Token（受信時のみ）
  is_push BOOLEAN DEFAULT FALSE, -- プッシュ通知かどうか（送信時のみ）
  related_table TEXT, -- 関連するテーブル名
  related_id TEXT, -- 関連するレコードID
  created_at TEXT NOT NULL,
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id)
);
```

**手動プッシュ通知履歴 (Manual Push Log)**

管理者が手動で送信したプッシュ通知の記録。

```sql
CREATE TABLE manual_push_log (
  id TEXT PRIMARY KEY,
  sent_by TEXT NOT NULL, -- 送信した管理者のID
  target_type TEXT NOT NULL, -- 'all', 'studio', 'recital', 'individual'
  target_ids JSON, -- 対象のID群（教室、発表会、個人など）
  message_content TEXT NOT NULL, -- 送信したメッセージ内容
  message_type TEXT NOT NULL, -- 'text', 'template', 'flex'
  sent_count INTEGER DEFAULT 0, -- 送信成功数
  failed_count INTEGER DEFAULT 0, -- 送信失敗数
  reason TEXT, -- 送信理由（緊急連絡、リマインダーなど）
  created_at TEXT NOT NULL
);
```

**キーワード応答設定 (Keyword Responses)**

LINE公式アカウントのキーワード応答設定。

```sql
CREATE TABLE keyword_responses (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL, -- テナント識別子
  keyword TEXT NOT NULL, -- 反応するキーワード（正規表現可）
  response_type TEXT NOT NULL, -- 'text', 'liff_link', 'template'
  response_content TEXT NOT NULL, -- 応答内容またはLIFF URL
  priority INTEGER DEFAULT 0, -- 優先度（高い方が優先）
  is_enabled BOOLEAN DEFAULT TRUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);
```

例:
- 「準備」→ 準備物一覧のLIFFリンクを返信
- 「確認」→ タスク確認のLIFFリンクを返信
- 「登録」→ 児童情報登録のLIFFリンクを返信

-----

## 7. セキュリティ設計

### 7.1 設計原則

子供のバレエ教室という性質上、**個人情報とプライバシーの保護**は最優先事項です。特に「いつ・誰が・どこにいるか」という情報は、悪意ある第三者に悪用される可能性があるため、厳重に管理します。

#### 基本方針

- **最小権限の原則**: 各ユーザーは自分の役割に必要な情報のみにアクセス可能
- **明示的な同意**: 情報共有には明確な同意を取得
- **監査可能性**: すべてのアクセスと変更を記録
- **デフォルトで非公開**: 情報は明示的に許可されない限り非公開

### 7.2 役割と権限

#### ユーザー役割の定義

| 役割 | 説明 | 権限レベル |
|------|------|-----------|
| 先生（owner） | 教室のオーナー | 全情報へのフルアクセス |
| 保護者代表（admin） | 運営を補助する保護者 | 生徒・レッスン情報の閲覧、連絡管理 |
| 一般保護者（guardian） | 通常の保護者 | 自分の子供に関する情報のみ |

#### 情報アクセス権限マトリクス

| 情報種別 | 先生 | 保護者代表 | 一般保護者 |
|---------|------|-----------|-----------|
| 自分の子供の情報 | ✅ | ✅（担当生徒のみ） | ✅ |
| 全生徒の名簿 | ✅ | ✅ | ❌ |
| 他の生徒の出欠状況 | ✅ | ✅ | ❌ |
| レッスン参加者一覧 | ✅ | ✅ | ❌ |
| 臨時レッスン対象者一覧 | ✅ | ✅ | ❌（自分の子供が対象かのみ） |
| 全体の応答状況集計 | ✅ | ✅ | ❌ |
| 発表会キャスティング | ✅ | ✅ | ✅（公開設定されたもの） |

### 7.3 レッスン情報のプライバシー保護

#### 臨時レッスンの情報公開制御

臨時レッスンは**対象となる生徒の保護者のみ**に通知されます。

```
[臨時レッスン: 発表会前特別練習]
├── 対象: 花子、桜子、太郎
│
├── 花子の保護者 → 「花子さんが対象の臨時レッスンがあります」✅
├── 桜子の保護者 → 「桜子さんが対象の臨時レッスンがあります」✅
├── 太郎の保護者 → 「太郎くんが対象の臨時レッスンがあります」✅
└── 他の生徒の保護者 → 通知なし、情報にアクセス不可 ❌
```

**設計ポイント**:
- 一般保護者は「誰が臨時レッスンに呼ばれたか」を知ることができない
- 自分の子供が対象かどうかのみ確認可能
- 参加/不参加の応答も本人のみ

#### 通常レッスンの出欠情報

通常レッスンの出欠情報も、一般保護者には自分の子供の情報のみ表示。

```
[一般保護者が見る画面]
花子さんの出欠状況
├── 12/16（月）通常レッスン: 出席
├── 12/18（水）通常レッスン: 欠席
└── 12/20（金）臨時レッスン: 参加予定

※ 他の生徒の出欠状況は表示されません
```

```
[先生・保護者代表が見る画面]
12/16（月）通常レッスン 出席状況
├── 出席: 花子、桜子、太郎、...（15名）
├── 欠席: 美咲、健太（2名）
└── 未回答: なし
```

### 7.4 応答情報の集計とプライバシー

#### 臨時レッスン応答の表示

| 閲覧者 | 表示される情報 |
|--------|---------------|
| 先生・保護者代表 | 参加者名一覧、不参加者名一覧、未回答者名一覧 |
| 一般保護者 | 自分の子供の応答状況のみ + 全体の集計数（例: 参加12名、不参加3名） |

**一般保護者への表示例**:
```
臨時レッスン: 発表会前特別練習
日時: 12/22（日）14:00-16:00

[花子さんの応答]
✅ 参加する

[全体の状況]
参加: 12名 / 不参加: 3名 / 未回答: 2名
（参加者名は表示されません）
```

#### 休校確認の表示

| 閲覧者 | 表示される情報 |
|--------|---------------|
| 先生・保護者代表 | 確認済み保護者一覧、未確認保護者一覧 |
| 一般保護者 | 自分の確認状況のみ |

### 7.5 データアクセス制御の実装

#### APIレベルでの制御

```
[APIエンドポイント設計]

GET /api/lessons/{id}/participants
├── 先生・保護者代表 → 全参加者リストを返却
└── 一般保護者 → 403 Forbidden（または自分の子供の情報のみ）

GET /api/lessons/{id}/responses
├── 先生・保護者代表 → 全応答リストを返却
└── 一般保護者 → 自分の子供の応答のみ返却

GET /api/lessons/{id}/summary
├── 全ユーザー → 集計情報のみ返却（参加X名、不参加Y名）
└── 個人を特定できる情報は含まない
```

#### クエリレベルでの制御

```sql
-- 一般保護者向け: 自分の子供のレッスンイベントのみ取得
SELECT le.*
FROM lesson_events le
WHERE le.id IN (
  -- 教室全体対象のイベント（自分の子供の教室）
  SELECT le2.id FROM lesson_events le2
  JOIN students s ON le2.studio = s.studio
  JOIN line_student_access lsa ON s.id = lsa.student_id
  WHERE lsa.line_account_id = :current_user_id
    AND le2.target_type = 'all_studio'

  UNION

  -- 特定生徒対象のイベント（自分の子供が対象）
  SELECT le3.id FROM lesson_events le3
  JOIN lesson_event_targets let ON le3.id = let.lesson_event_id
  JOIN line_student_access lsa2 ON let.student_id = lsa2.student_id
  WHERE lsa2.line_account_id = :current_user_id
);
```

### 7.6 セキュリティ監査

#### アクセスログの記録

すべての機密情報へのアクセスを記録し、不正アクセスを検知。

```sql
CREATE TABLE access_log (
  id TEXT PRIMARY KEY,
  line_account_id TEXT NOT NULL,
  resource_type TEXT NOT NULL, -- 'lesson_event', 'student', 'response', etc.
  resource_id TEXT NOT NULL,
  action TEXT NOT NULL, -- 'view', 'list', 'export'
  access_granted BOOLEAN NOT NULL, -- アクセスが許可されたか
  ip_address TEXT,
  user_agent TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY (line_account_id) REFERENCES line_accounts(id)
);
```

#### 異常検知

- 短時間での大量アクセス
- 権限外の情報へのアクセス試行
- 通常と異なるパターンのアクセス

### 7.7 通知におけるプライバシー配慮

#### LINE通知の内容制限

通知メッセージには**最小限の情報のみ**を含め、詳細はLIFFアプリで確認する設計。

**良い例**:
```
📢 臨時レッスンのお知らせ
花子さん宛の臨時レッスンがあります。
詳細を確認 → [LIFFリンク]
```

**悪い例**（避けるべき）:
```
📢 臨時レッスンのお知らせ
12/22（日）14:00-16:00 ○○スタジオ
発表会前特別練習
対象: 花子、桜子、太郎、美咲、...
```

#### 理由

- LINEメッセージはスマホのロック画面に表示される可能性
- 第三者に見られても個人の行動情報が漏れないように
- 詳細は認証済みのLIFFアプリ内でのみ表示

-----

## 8. 多言語対応戦略

### 8.1 翻訳対象の選別

#### 必須翻訳（高品質な翻訳が必要）

**対象**:

- 先生からの説明・指示文（演目の解釈、衣装の仕様、変更理由など）
- 保護者からのコメント・質問（「サイズが合わない」「作り方がわからない」など）
- 重要な通知メッセージ（日程変更、至急の確認依頼など）

**理由**:

- ニュアンスの誤解が準備に支障をきたす
- 細かい表現の違いが重要

**手法**:

- TPOコンテキストを活用した高精度翻訳
- バレエ用語辞書の参照
- 編集時に事前生成、DBに保存

#### 英語統一（翻訳不要）

**対象**:

- ステータス表記: `Pending` / `In Progress` / `Completed` / `Issue`
- カテゴリ・ラベル: `Costume` / `Props` / `Backstage Tasks`
- UIアクション: `Save` / `Cancel` / `Edit` / `Delete`
- 日付・時刻: ISO 8601形式（`2024-12-25 18:00`）

**理由**:

- 単語レベルで意味が明確
- UIで視覚的にも判別可能（アイコン併用）
- 情報の一貫性が向上（解釈のブレがない）
- データ容量・翻訳コストの削減

#### 原語維持（参考訳は任意）

**対象**:

- 演目名: `Swan Lake`, `The Nutcracker`
- 役名: `Odette`, `Clara`
- 専門用語: `Grand Jeté`, `Arabesque`

**理由**:

- バレエ作品の国際的な固有名詞
- 翻訳すると逆に混乱を招く
- 必要に応じて参考訳を併記（例: `Swan Lake（白鳥の湖）`）

### 8.2 翻訳生成のタイミング

**編集時に事前生成**

```
[管理者が説明文を入力/更新]
    ↓
[Workers AI で翻訳を生成]
    ↓ (TPOコンテキスト + 用語辞書を付与)
[translations テーブルに保存]
    ↓
[閲覧時は DB から取得]
```

**メリット**:

- 同じテキストを何度も翻訳しない（コスト削減）
- 翻訳結果が一貫する（ユーザーごとのブレがない）
- 閲覧時のレスポンス高速化
- 翻訳精度向上のためのコンテキスト活用が容易

**再翻訳**:

- 管理画面に「翻訳を再生成」ボタンを提供
- 用語辞書更新時に一括再翻訳も可能

### 8.3 TPOコンテキストの活用

翻訳時にAIへ提供するコンテキスト情報。

#### Time（時）

- 通常連絡 / 緊急連絡 / リマインダー
- 期日情報の有無

#### Place（場所）

- レッスン / リハーサル / 本番 / 準備作業
- 対象の教室・会場

#### Occasion（場合）

- 対象読者: 先生 / 保護者 / 生徒
- 目的: 指示 / 通知 / 依頼 / 確認
- 緊急度: 通常 / 注意 / 至急
- 専門領域: 技術 / 衣装 / 事務 / 一般

#### 翻訳プロンプト例

```
Context:
- Time: Urgent notice with deadline 2024-12-20
- Place: Costume preparation for recital
- Occasion: Request to parents, High urgency
- Domain: Ballet costume

Glossary:
- tutu: チュチュ (ballet skirt)
- romantic tutu: ロマンティックチュチュ (longer, softer tutu)

Translate to Japanese:
"Please check if the romantic tutu fits your daughter by December 20. 
If there are any size issues, let us know immediately."
```

期待される出力:

```
「12月20日までにロマンティックチュチュ（長めのふんわりしたスカート）が
お嬢様に合うかご確認ください。サイズに問題がある場合は至急お知らせください。」
```

### 8.4 データ容量の比較

**全翻訳方式**:

```
description.ja: "ふんわりとしたロマンティックチュチュ" (48 bytes)
description.en: "Soft romantic tutu" (18 bytes)
description.ru: "Мягкая романтическая пачка" (52 bytes)
status.ja: "準備中" (9 bytes)
status.en: "In Progress" (11 bytes)
status.ru: "В процессе" (20 bytes)
合計: 158 bytes
```

**選択的翻訳方式**:

```
description.ja/en/ru: 118 bytes (同上)
status: "In Progress" (11 bytes) ← 英語統一
合計: 129 bytes
```

**削減効果**:

- 1アイテムあたり 29 bytes 削減 (18%減)
- 100アイテムで 2.9KB 削減
- 翻訳APIコールも約40%削減

-----

## 9. 画面構成

### 9.1 管理者画面

#### ダッシュボード

- 発表会の進捗概要
  - マイルストーンの達成状況
  - タスク完了率（進捗バー）
  - 優先度別の未完了タスク数
- 未対応事項の一覧
  - 期日が近いタスク
  - ブロッカーが報告されているタスク
  - 未解決の質問
- 最近の更新履歴（監査ログから）
- クリティカルパスのハイライト

#### 生徒・LINEアカウント管理

- 生徒情報の登録、編集、削除
- **アクセスコードの管理**（発行、再発行、無効化）
- パスキー発行・無効化（LINE未使用者向け）
- **生徒とLINEアカウントの紐づけ一覧**
  - どの生徒にどのLINEアカウントが紐づいているか
  - 関係（父・母・祖父母など）の確認
  - 主担当（is_primary）の確認・変更
- **LINE連携状況の確認**（友だち追加済み、登録完了など）
- **紐づけの手動追加・削除**（管理者による強制紐づけ）

#### レッスン管理

**スケジュール管理**:

- 通常レッスンの定期スケジュール設定（曜日・時間・教室・クラス）
- カレンダー表示（月表示・週表示）
- 祝日・長期休暇の設定

**臨時レッスン管理**:

- 臨時レッスンの作成
  - 日時・場所の設定
  - 対象の選択（教室全体 / 特定クラス / 選択された生徒）
  - 応答締切の設定
- 臨時レッスンの一覧表示
- 応答状況の確認（参加/不参加/未回答）
- 未回答者へのリマインダー送信

**休校管理**:

- 休校の登録（日時・対象・理由）
- 振替レッスンの設定
- 確認状況の追跡

**出欠管理**:

- レッスンごとの出欠一覧
- 出欠統計（生徒別・期間別）
- 欠席連絡の確認

#### 発表会管理

- 発表会の作成・編集
- **テンプレートからの作成**（過去の発表会をベースに）
- 演目の登録、順番の変更
- **マイルストーンの設定**
- キャスティング設定（演目×生徒×役名）
- 確定/未確定の切り替え

#### 準備物・タスク管理

**多角的なビュー**:

- **リストビュー**: 詳細な情報の一覧表示
- **カンバンビュー**: ステータス別のカード表示（ドラッグ&ドロップ対応）
- **タイムラインビュー**: 期日ベースのガントチャート風表示
- **依存関係図**: タスク間の関係性を可視化

**タスク管理機能**:

- 準備物・タスクの作成（type, category, attributes を指定）
- **優先度の設定**（Critical / High / Medium / Low）
- **担当者・確認者の割り当て**
- **依存関係の設定**
- **完了条件（Definition of Done）の定義**
- 演目・役・生徒との関連付け
- ステータス定義のカスタマイズ
- 全体の進捗確認

**ブロッカー管理**:

- ブロッカーの一覧
- 解決状況の追跡
- ブロックされているタスクのハイライト

#### コミュニケーション

- 保護者からのコメント・質問の確認
- 返信・解決済みマーク
- 問い合わせの多い項目の可視化

#### 翻訳管理

- 翻訳の再生成
- 用語辞書の編集
- 翻訳履歴の確認

#### LINE公式アカウント管理

- **リッチメニューの設定**
- **キーワード応答の設定**（キーワードと応答内容の管理）
- **ウェルカムメッセージの設定**
- LINEメッセージ履歴の閲覧

#### 手動プッシュ通知

- **手動プッシュ通知の送信**（緊急時のみ使用）
  - 送信対象の選択（全員、教室別、発表会別、個人）
  - メッセージ内容の作成
  - 送信理由の記録
- プッシュ通知の送信履歴
- 通知テンプレートの編集

#### テンプレート管理

- テンプレートの作成・編集
- 発表会からテンプレートを生成
- テンプレート項目の管理

#### 変更履歴

- 監査ログの閲覧
- フィルタリング（期間、変更者、テーブル）
- 変更内容の詳細表示

### 9.2 保護者画面（LIFF アプリ）

LINE公式アカウントのリッチメニューからアクセスするLIFFアプリとして提供。

#### アクセス方法

1. **リッチメニュー**: LINE公式アカウントのトーク画面下部に常時表示
2. **キーワード応答**: 特定のキーワードを送信するとLIFFリンクを返信
3. **Webブラウザ直接アクセス**: LINE未使用者向け（パスキー認証）

#### 初回登録画面

LIFFアプリ初回起動時に表示。2つのモードを選択可能。

**新規登録モード（最初の保護者）**:
- 児童情報の登録（名前、教室）
- 自分と児童の関係を選択（母/父/祖父母/その他）
- 希望言語の選択
- LINE User IDと自動紐づけ
- **アクセスコードの表示**（他の保護者と共有用）

**アクセスコード入力モード（追加の保護者）**:
- アクセスコードを入力
- 自分と児童の関係を選択
- 希望言語の選択
- 既存の生徒に自分のLINEアカウントを紐づけ

#### 子供を追加画面

既に1人以上の子供を登録済みの保護者向け。

**新規登録（兄弟姉妹の追加）**:
- 新しい児童情報の登録
- アクセスコードが自動発行される

**アクセスコード入力（既存の子供に追加）**:
- 他の保護者が登録済みの子供に紐づける場合
- アクセスコードを入力して紐づけ

#### アクセスコード管理

登録済みの子供のアクセスコードを確認・共有。

- 子供ごとのアクセスコード表示
- **LINEで共有**ボタン（アクセスコードをメッセージで送信）
- アクセスコードの再発行（古いコードは無効化）
- 紐づけ済みの保護者一覧の確認

#### マイページ（ホーム）

- 自分の子供に関する情報を一覧表示
- **今後のレッスン予定**（通常レッスン・臨時レッスン）
- **要対応事項**（臨時レッスンへの応答、休校確認など）
- 発表会の日程・会場
- 出演する演目・役（確定/未確定）
- 準備物のチェックリスト
- **マイルストーンまでの残り日数**
- **優先度の高いタスクのハイライト**
- **未読の更新通知**

#### レッスンスケジュール

**カレンダー表示**:
- 自分の子供のレッスン予定を表示
- 通常レッスン（定期）
- 臨時レッスン（対象の場合のみ表示）
- 休校（対象の場合のみ表示）

**レッスン詳細**:
- 日時・場所・クラス
- 臨時レッスンの場合は説明文
- 応答状況（自分の子供のみ）

#### 臨時レッスン応答

**臨時レッスン一覧**:
- 自分の子供が対象の臨時レッスンのみ表示
- 応答締切の表示
- 応答状況（参加予定/不参加/未回答）

**応答画面**:
- 参加/不参加の選択
- 備考の入力（任意）
- 応答の変更（締切前のみ）

**プライバシー保護**:
- 他の参加者の名前は表示されない
- 全体の集計数のみ表示（例: 参加12名、不参加3名）

#### 休校確認

**休校連絡一覧**:
- 自分の子供に関係する休校連絡のみ表示
- 確認状況（確認済み/未確認）

**確認画面**:
- 休校の詳細（日時・理由）
- 振替情報（ある場合）
- 「確認しました」ボタン

#### 欠席連絡

**欠席連絡の送信**:
- レッスンを選択
- 対象の子供を選択（複数いる場合）
- 理由の入力（任意）
- 送信確認

**欠席連絡履歴**:
- 送信済みの欠席連絡一覧
- ステータス（先生確認済み/未確認）

#### 発表会詳細

- 日程、会場、プログラム（全演目の一覧）
- 自分の子供が出演する演目をハイライト
- 確定/未確定の明示
- **マイルストーンの一覧と進捗**

#### 準備物一覧

**ビュー切替**:
- リスト表示（デフォルト）
- カンバン表示（ステータス別）
- タイムライン表示（期日順）

**機能**:
- 衣装・小道具のチェックリスト
- ステータス更新（ドロップダウン選択）
- 入手方法の選択（購入/手作り/貸出/手持ち）
- コメント投稿（質問・相談）
- **ブロッカーの報告**（「〇〇が理由で進められない」）
- **完了条件の確認**

#### 担当タスク

- 裏方タスクの確認
- ステータス更新
- コメント投稿
- **依存関係の確認**（何を待っているか、何が待っているか）

#### 設定

- 希望言語の変更（日本語 / 英語 / ロシア語）
- 児童情報の編集

#### 変更履歴

- 自分に関連する変更の履歴
- 「最後に見てから何が変わったか」の表示

### 9.3 LINEメッセージ応答

LINE公式アカウントへのメッセージ送信時の自動応答（Reply Token使用）。

#### キーワード応答例

| キーワード | 応答内容 |
|-----------|---------|
| 準備 | 準備物一覧のLIFFリンク |
| タスク | 担当タスク一覧のLIFFリンク |
| 確認 | マイページのLIFFリンク |
| 登録 | 児童情報登録画面のLIFFリンク |
| レッスン | 今後のレッスン予定のLIFFリンク |
| 臨時 | 未回答の臨時レッスン一覧のLIFFリンク |
| 休校 | 休校情報のLIFFリンク |
| 欠席 | 欠席連絡画面のLIFFリンク |
| ヘルプ | 使い方の説明テキスト |

#### 未登録ユーザーへの応答

児童情報未登録のユーザーには、登録画面への導線を優先的に案内。

```
ご利用ありがとうございます。
まずはお子様の情報を登録してください。
→ [登録はこちら]
```

-----

## 10. 通知設計

### 10.1 基本方針

**プル型を基本とし、プッシュは最後の手段**

本システムでは、コスト削減と運用負荷軽減のため、**プル型（保護者が自ら情報を取得）** を基本とし、**プッシュ通知は緊急時のみ手動で送信** する設計とする。

| 通知方式 | 説明 | コスト |
|---------|------|--------|
| Reply Token応答 | 保護者がメッセージを送信した際に返信 | **無料** |
| プッシュ通知 | 管理者が手動で送信（緊急時のみ） | 有料（月200通まで無料） |

**設計思想**:

- 保護者は必要な時にLIFFアプリを開いて情報を確認（プル型）
- キーワードを送信すれば、Reply Tokenで最新情報を返信（無料）
- 緊急時のみ管理者が判断してプッシュ通知を送信

### 10.2 Reply Token応答（無料）

保護者がLINE公式アカウントにメッセージを送信した際、Reply Tokenを使って即座に応答。

#### 応答パターン

**ステータス確認**

保護者: 「確認」
```
📋 お子様の準備状況

[花子さん]
✅ 衣装（白チュチュ）: 完了
⏳ 小道具（花冠）: 準備中
❌ リボン: 未着手

詳細はこちら → [LIFFリンク]
```

**未完了タスク一覧**

保護者: 「タスク」
```
📝 未完了のタスク

1. リボンの購入（12/20まで）
2. 靴のサイズ確認（12/22まで）

詳細・ステータス更新 → [LIFFリンク]
```

**最新のお知らせ**

保護者: 「お知らせ」
```
📢 最新のお知らせ

[12/15] 発表会の集合時間が変更になりました
[12/14] 衣装の最終確認について

詳細 → [LIFFリンク]
```

**レッスン予定確認**

保護者: 「レッスン」
```
📅 今後のレッスン予定

[花子さん]
12/18（水）16:00 通常レッスン
12/20（金）14:00 臨時レッスン ⚠️要回答
12/22（日）休校

詳細・応答 → [LIFFリンク]
```

**臨時レッスン応答**

保護者: 「臨時」
```
📢 要対応の臨時レッスン

[花子さん]
12/20（金）14:00 発表会前特別練習
締切: 12/18（水）まで
現在の回答: 未回答

応答する → [LIFFリンク]
```

**休校確認**

保護者: 「休校」
```
📢 休校のお知らせ

[花子さん]
12/22（日）台風接近のため休校
振替: 12/29（日）14:00

確認済みにする → [LIFFリンク]
```

### 10.3 手動プッシュ通知（最後の手段）

プッシュ通知は**緊急時のみ**管理者が手動で送信。自動送信は行わない。

**プッシュ通知を使用するケース**:

| ケース | 例 |
|--------|-----|
| 緊急の日程変更 | 「本日のリハーサルが中止になりました」 |
| 重要な締め切り直前 | 「衣装サイズ確認の締め切りが明日です」 |
| 全員への重要連絡 | 「発表会会場が変更になりました」 |
| 当日の休校連絡 | 「本日は台風のため休校です」 |
| 臨時レッスンの緊急追加 | 「明日、臨時レッスンを行います」 |

**プッシュ通知を使用しないケース**:

- 通常の情報更新（保護者がLIFFで確認）
- 定期的なリマインダー（保護者がキーワードで確認）
- 個別の質問への返信（Reply Tokenで応答）

#### プッシュ通知の送信フロー

1. 管理者が管理画面で「手動プッシュ通知」を選択
2. 送信対象を選択（全員 / 教室別 / 発表会別 / 個人）
3. メッセージ内容を作成
4. 送信理由を記録（監査のため）
5. 確認画面で送信数とコストを確認
6. 送信実行

#### プッシュ通知のテンプレート例

**緊急連絡**:
```
🚨 緊急のお知らせ

[内容]

詳細 → [LIFFリンク]
```

**リマインダー**:
```
⏰ リマインダー

[締め切り情報]

確認・更新 → [LIFFリンク]
```

**休校連絡**:
```
📢 休校のお知らせ

花子さんのレッスンについて
休校のお知らせがあります。

詳細を確認 → [LIFFリンク]
```

**臨時レッスン連絡**:
```
📢 臨時レッスンのお知らせ

花子さん宛の臨時レッスンがあります。
参加/不参加をお知らせください。

詳細・応答 → [LIFFリンク]
```

**注意**: プライバシー保護のため、通知メッセージには日時・場所・他の参加者などの詳細情報を含めません。詳細はLIFFアプリ内でのみ表示します。

### 10.4 アプリ内通知（LIFFアプリ）

LIFFアプリ内での通知表示。プッシュ通知なしでも更新を把握可能。

**未読管理**:

- マイページに未読件数をバッジ表示
- 「最後にアクセスしてからの変更」をハイライト
- 更新された項目に「NEW」マークを表示

**更新履歴の表示**:

- 自分に関連する変更の履歴
- 時系列での変更一覧
- フィルタリング（種類、期間）

### 10.5 通知の多言語化

応答メッセージは保護者の希望言語に応じて送信。

**日本語**:
```
📋 お子様の準備状況
✅ 衣装: 完了
詳細 → [リンク]
```

**英語**:
```
📋 Your child's preparation status
✅ Costume: Completed
Details → [link]
```

**ロシア語**:
```
📋 Статус подготовки вашего ребенка
✅ Костюм: Готов
Подробнее → [ссылка]
```

### 10.6 コスト比較

**従来のプッシュ通知方式**:

| 項目 | 月間通知数 | コスト |
|------|-----------|--------|
| 定期リマインダー | 100通 | 有料 |
| 更新通知 | 200通 | 有料 |
| 緊急通知 | 10通 | 有料 |
| **合計** | 310通 | 月額数百円〜 |

**本システム（Reply Token + 手動プッシュ）**:

| 項目 | 月間通知数 | コスト |
|------|-----------|--------|
| Reply Token応答 | 無制限 | **無料** |
| 手動プッシュ（緊急時のみ） | 〜50通 | 無料枠内 |
| **合計** | - | **月額0円** |

-----

## 11. 開発計画

### Phase 1: 基盤構築

1. Cloudflare Workers + D1 のセットアップ
1. データベーススキーマの作成
1. 認証機能（パスワード認証：管理者向け）の実装
1. 基本的な管理画面のUI構築

**成果物**:

- ログイン可能な管理システム
- 生徒・保護者の登録機能

### Phase 2: LINE Platform 構築

1. LINE公式アカウントの開設と設定
1. LIFFアプリの作成と登録
1. LINE Login連携の実装
1. Webhook受信とReply Token応答の実装
1. リッチメニューの設定

**成果物**:

- LINE公式アカウント（友だち追加可能）
- LIFFアプリ（初回登録画面）
- キーワード応答の基本実装

### Phase 3: コア機能

1. 発表会・演目・キャスティング管理
1. 準備物・タスク管理とステータス追跡
1. LIFFアプリでの保護者向け画面（マイページ、準備物一覧）
1. 双方向コミュニケーション機能（コメント・質問）
1. 手動プッシュ通知機能

**成果物**:

- 発表会の情報を管理・閲覧できるシステム
- LIFFアプリでの準備物ステータス追跡
- 管理者向け手動プッシュ通知画面

### Phase 4: 多言語対応

1. Workers AI による翻訳機能
   - TPOコンテキストの実装
   - 用語辞書の整備
1. Reply Token応答の多言語化
1. LIFFアプリの多言語対応

**成果物**:

- 多言語対応の完成
- 言語別Reply Token応答

### Phase 5: 運用と改善（継続）

1. 次回発表会での試験運用
1. フィードバックに基づく改善
1. キーワード応答パターンの拡充
1. ドキュメント整備
1. ユーザーマニュアルの作成

**成果物**:

- 本番運用可能なシステム
- 運用マニュアル
- LINE公式アカウント運用ガイド

-----

## 12. 技術スタック

|レイヤー   |技術                        |
|-------|--------------------------|
|ホスティング |Cloudflare Workers        |
|データベース |Cloudflare D1 (SQLite)    |
|AI     |Cloudflare Workers AI (翻訳)|
|フロントエンド（管理者）|Hono + JSX または React      |
|フロントエンド（保護者）|LIFF（LINE Front-end Framework）|
|LINE Platform|LINE公式アカウント + LINE Login + Messaging API|
|認証     |LINE Login（保護者）、パスキー（LINE未使用者）、パスワード（管理者）|

### LINE Platform 構成詳細

| コンポーネント | 用途 |
|--------------|------|
| LINE公式アカウント | 保護者との接点、リッチメニュー、Webhook受信 |
| LIFF | LINE内で動作するWebアプリ（保護者向けUI） |
| LINE Login | LIFFアプリでの認証、LINE User ID取得 |
| Messaging API | Reply Token応答、手動プッシュ通知 |

-----

## 13. 運用コスト

Cloudflare の無料枠および低コストプランを活用。LINE連携はReply Token中心でコストを最小化。

|サービス              |コスト（概算）                    |
|------------------|---------------------------|
|Cloudflare Workers|無料枠で十分（10万リクエスト/日）         |
|Cloudflare D1     |無料枠で十分（5GB、500万行読み取り/日）    |
|Workers AI        |無料枠 + 従量課金（翻訳は編集時のみ、月数百円程度）|
|LINE公式アカウント（フリープラン）|無料|
|LINE Messaging API（Reply Token）|**無料（無制限）**|
|LINE Messaging API（プッシュ）|月200通まで無料（緊急時のみ使用）|
|**合計**            |**月額 0円〜数百円**               |

**コスト削減のポイント**:

- リアルタイム翻訳を避け、編集時に事前生成
- 翻訳対象を選別（必要なものだけ翻訳）
- Cloudflare の無料枠を最大限活用
- **Reply Token応答を活用し、プッシュ通知を最小化**
- **プル型設計により自動通知を廃止**

-----

## 14. 設計課題

本セクションでは、実装に向けて詳細設計が必要な課題を整理する。

### 14.1 マルチテナント関連

#### 14.1.1 テナントオンボーディング

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| テナント作成フロー | 新規テナントの申込→審査→作成→初期設定の一連のフロー | 高 |
| LINE公式アカウント連携 | テナントごとのLINE公式アカウント開設とシステム連携の手順 | 高 |
| 初期データ投入 | 教室・クラス・生徒情報の一括インポート方法 | 中 |
| 管理者招待 | テナントオーナーが他の管理者を招待する仕組み | 中 |

#### 14.1.2 テナント分離の詳細

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| クエリフィルタの強制 | 全クエリに tenant_id フィルタを確実に適用するミドルウェア設計 | 高 |
| テナント越境検知 | 不正なテナント越境アクセスの検知と通知 | 高 |
| 共有リソースの扱い | システム共通テンプレート・用語辞書のアクセス制御 | 中 |
| データエクスポート | テナント単位でのデータエクスポート機能 | 低 |

#### 14.1.3 課金・プラン管理

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| プラン定義 | free / basic / premium 各プランの機能制限の詳細 | 中 |
| 利用量計測 | 生徒数、管理者数、翻訳回数などの計測方法 | 中 |
| 請求処理 | 課金システムとの連携（Stripe等） | 低 |
| プラン変更 | アップグレード/ダウングレード時のデータ処理 | 低 |

### 14.2 認証・認可関連

#### 14.2.1 LINE認証の詳細

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 複数テナント所属 | 同一LINEユーザーが複数テナントに所属する場合のUI/UX | 高 |
| テナント切替 | 保護者が複数教室を利用する場合のテナント切替方法 | 高 |
| LIFF初回起動 | テナント未登録ユーザーのLIFF初回起動時のフロー | 高 |
| セッション管理 | テナントコンテキストを含むセッション管理方式 | 高 |

#### 14.2.2 管理者認証

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| パスワードポリシー | パスワード強度要件、有効期限 | 中 |
| 二要素認証 | 管理者向け2FAの導入可否 | 低 |
| ログイン履歴 | 管理者ログイン履歴の保存と監査 | 中 |

### 14.3 データモデル関連

#### 14.3.1 未定義の詳細

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 教室マスタ | studio カラムの値をテナント設定で定義する仕組み | 高 |
| クラスマスタ | class_name の管理方法（マスタテーブル or 自由入力） | 中 |
| ステータス定義 | テナントごとにカスタマイズ可能なステータス定義の管理 | 中 |
| 削除ポリシー | 論理削除 vs 物理削除の方針、データ保持期間 | 中 |

#### 14.3.2 マイグレーション

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| スキーマバージョン管理 | D1でのマイグレーション管理方法 | 高 |
| 後方互換性 | スキーマ変更時の既存データへの影響 | 高 |
| ロールバック | 問題発生時のロールバック手順 | 中 |

### 14.4 LINE連携関連

#### 14.4.1 Webhook処理

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 署名検証 | テナントごとのChannel Secretを使った署名検証 | 高 |
| リトライ対応 | LINE Webhook のリトライ時の冪等性確保 | 高 |
| エラー通知 | Webhook処理エラー時の管理者通知 | 中 |

#### 14.4.2 LIFF設定

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| LIFF URLの構造 | テナントごとのLIFF URL設計（1 LIFF vs テナント別LIFF） | 高 |
| Endpoint URL | LIFF Endpoint URLの動的設定方法 | 高 |
| 権限スコープ | 必要なLINE Login権限スコープの整理 | 中 |

### 14.5 翻訳機能関連

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 翻訳キャッシュ | 同一テキストの再翻訳を避けるキャッシュ戦略 | 中 |
| 翻訳品質監視 | 翻訳結果の品質確認と手動修正の仕組み | 低 |
| 用語辞書の共有 | システム共通辞書とテナント固有辞書のマージ方法 | 中 |
| コスト管理 | テナントごとの翻訳API使用量の計測と制限 | 中 |

### 14.6 運用・保守関連

#### 14.6.1 監視・アラート

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| ヘルスチェック | システム正常性の監視方法 | 高 |
| エラー通知 | 重大エラー発生時の運用者への通知 | 高 |
| パフォーマンス監視 | レスポンスタイム、D1クエリ性能の監視 | 中 |

#### 14.6.2 バックアップ・復旧

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| バックアップ戦略 | D1のバックアップ頻度と保持期間 | 高 |
| 障害復旧手順 | データ破損時の復旧手順 | 高 |
| テナント削除 | テナント解約時のデータ削除手順とアーカイブ | 中 |

### 14.7 セキュリティ関連

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 機密情報の暗号化 | Channel Secret等の機密情報の保存方法 | 高 |
| アクセスコードの強度 | アクセスコード（例: ABC-1234）の衝突確率と強度 | 中 |
| レート制限 | API呼び出しのレート制限の実装 | 中 |
| CORS設定 | 許可するオリジンの管理方法 | 中 |

### 14.8 今後の拡張に向けた課題

| 課題 | 詳細 | 優先度 |
|------|------|--------|
| 複数言語追加 | 日英露以外の言語追加時の影響範囲 | 低 |
| 他プラットフォーム対応 | LINE以外（WhatsApp, WeChat等）への対応可能性 | 低 |
| ネイティブアプリ | 将来的なネイティブアプリ化の可能性と設計考慮 | 低 |
| 分析・レポート | 出欠統計、準備進捗レポート等の分析機能 | 低 |

-----

## 15. 期待される効果

### 15.1 情報の可視化

- 「何が決まっているか分からない」問題の解消
- 情報が一元化され、最新状態が常に参照可能
- 決定事項と未定事項の明確な分離
- **変更履歴による透明性の確保**

### 15.2 言語の壁の解消

- 日本語・英語・ロシア語での情報アクセス
- 高精度な翻訳（TPOコンテキスト + 用語辞書）
- 全保護者が同等の情報にアクセス

### 15.3 準備物の抜け漏れ防止

- ステータス追跡による進捗の可視化
- 双方向コミュニケーションによる問題の早期発見
- 期日管理とリマインダー
- **依存関係の明示による作業順序の明確化**
- **完了条件による認識ずれの防止**

### 15.4 統括者の負担軽減

- 「分からない」と答えるしかなかった状況から脱却
- システムを参照すれば答えられる状態へ
- 先生への質問集中を緩和
- **ブロッカーの可視化による問題の早期対応**

### 15.5 芸術的判断の尊重

- 直前までの変更に対応できる柔軟なシステム設計
- 確定/未確定の明示により、保護者も心の準備が可能
- 「完成度を妥協しない」方針を支援

### 15.6 プロジェクト管理の成熟

- **マイルストーンによる中間目標の明確化**
- **テンプレートによる知見の蓄積と再利用**
- **カンバン/タイムラインによる多角的な進捗把握**
- **通知のパーソナライズによる適切な情報配信**

### 15.7 継続的な改善

- 監査ログによる振り返りの材料
- テンプレートへのベストプラクティス蓄積
- 発表会ごとの改善サイクル

### 15.8 サービス横展開

- **マルチテナント設計**による複数教室の効率的運営
- 新規テナント追加が容易（データ登録のみ）
- 共通テンプレート・用語辞書の再利用
- 運用コストの分散による持続可能なサービス提供

-----

## 付録

### A. 用語集

#### システム固有の用語

|用語        |説明                                                 |
|----------|---------------------------------------------------|
|パスキー      |LINE未使用者向けの認証キー。`ballet-sakura-7842` のような形式             |
|TPO       |Time（時）、Place（場所）、Occasion（場合）の略。翻訳精度向上のためのコンテキスト情報|
|Workers AI|Cloudflare が提供するエッジAIサービス。翻訳などに利用                  |
|D1        |Cloudflare が提供するサーバーレスSQLiteデータベース                 |

#### LINE Platform 用語

|用語        |説明                                                 |
|----------|---------------------------------------------------|
|LINE公式アカウント|企業・団体がLINE上で運営するビジネス用アカウント。友だち追加した利用者とコミュニケーション可能|
|LIFF      |LINE Front-end Framework。LINE内で動作するWebアプリケーションを構築するためのフレームワーク|
|LINE Login|LINEアカウントを使った認証機能。LIFFアプリではLINE User IDを自動取得可能|
|Reply Token|LINEメッセージ受信時に発行されるトークン。このトークンを使った返信は**無料**|
|プッシュ通知   |ユーザーのアクションなしに送信する通知。**有料**（月200通まで無料）|
|Webhook   |LINEからサーバーへのイベント通知。メッセージ受信時などに呼び出される|
|リッチメニュー  |LINE公式アカウントのトーク画面下部に表示されるメニュー。LIFFアプリへの導線に利用|
|Flex Message|柔軟なレイアウトでカスタマイズ可能なLINEメッセージ形式|

#### プロジェクト管理用語

|用語        |説明                                                 |
|----------|---------------------------------------------------|
|マイルストーン   |プロジェクトの中間チェックポイント。達成すべき重要な節目                      |
|依存関係      |タスク間の「AはBの完了後に開始」という関係性                           |
|ブロッカー     |タスクの進行を妨げる問題や障害                                    |
|クリティカルパス |プロジェクト完了までの最長経路。遅延が全体に影響するタスクの連鎖                  |
|Definition of Done|完了条件。タスクが「完了」とみなされるための明確な基準                     |
|RACI      |Responsible（実行者）、Accountable（責任者）、Consulted（相談先）、Informed（通知先）の略|
|カンバン      |ステータス別にタスクを可視化する手法。ボード形式で進捗を管理                    |
|ガントチャート   |タスクを時間軸で可視化する図表。期日と依存関係を表示                        |
|WIP制限     |Work In Progress制限。同時進行タスク数の上限を設けてフローを改善           |
|テンプレート    |再利用可能なタスクパターン。過去の知見を蓄積                            |
|監査ログ      |すべての変更を記録する履歴。透明性と説明責任の確保                         |

### B. 参考資料

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 Documentation](https://developers.cloudflare.com/d1/)
- [Cloudflare Workers AI Documentation](https://developers.cloudflare.com/workers-ai/)
- [LINE Messaging API Documentation](https://developers.line.biz/ja/docs/messaging-api/)
- [LIFF Documentation](https://developers.line.biz/ja/docs/liff/)
- [LINE Login Documentation](https://developers.line.biz/ja/docs/line-login/)
- [LINE公式アカウント](https://www.linebiz.com/jp/service/line-official-account/)

-----

**以上**