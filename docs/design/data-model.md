# データモデル設計

**作成日**: 2024年12月
**更新日**: 2025年12月

---

## 6.1 設計原則

**柔軟性を重視**

- 固定的なエンティティを避ける
- カテゴリ・ステータスは管理者が定義
- 属性は JSON 形式で拡張可能
- 多対多の関連付けで柔軟性を確保

## 6.2 主要エンティティ

### 6.2.1 基本情報

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

### 6.2.2 レッスン管理

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

### 6.2.3 発表会管理

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

### 6.2.4 マイルストーン管理

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

### 6.2.5 準備物・タスク管理

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

### 6.2.6 翻訳管理

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

### 6.2.7 バレエ用語辞書

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

### 6.2.8 監査ログ・変更履歴

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

### 6.2.9 通知設定

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

### 6.2.10 テンプレート機能

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

### 6.2.11 LINE連携

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
