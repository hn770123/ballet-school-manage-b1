# バレエ教室運営支援システム 基本設計書

**作成日**: 2024年12月  
**バージョン**: 1.0

-----

## 目次

1. [システム概要](#1-システム概要)
1. [背景と課題](#2-背景と課題)
1. [設計思想](#3-設計思想)
1. [システム構成](#4-システム構成)
1. [認証設計](#5-認証設計)
1. [データモデル設計](#6-データモデル設計)
1. [多言語対応戦略](#7-多言語対応戦略)
1. [画面構成](#8-画面構成)
1. [通知設計](#9-通知設計)
1. [開発計画](#10-開発計画)
1. [技術スタック](#11-技術スタック)
1. [運用コスト](#12-運用コスト)

-----

## 1. システム概要

子供バレエ教室における発表会運営の課題を解決する多言語対応Webシステム。

### 1.1 コアバリュー

- **情報の可視化**: 「何が決まっていて何が決まっていないか」を明確化
- **芸術的判断の尊重**: 直前までの変更に対応できる柔軟性
- **言語の壁の解消**: 日本語・英語・ロシア語での情報アクセス
- **双方向コミュニケーション**: 保護者からのステータス報告・問い合わせの構造化

### 1.2 主要機能

- 発表会・演目・キャスティング管理
- 準備物（衣装・小道具）のステータス追跡
- 裏方タスクの管理と割り当て
- 多言語対応（選択的翻訳方式）
- LINE通知連携

-----

## 2. 背景と課題

### 2.1 教室の現状

- 3教室を運営（専用スタジオではなく間借り）
- 先生はポーランド人（日本語会話可、読み書き困難）
- 生徒は日本人中心、ロシア人家庭も含む
- コミュニケーションツールはLINEグループチャットのみ

### 2.2 解決すべき課題

#### 課題1: コミュニケーションの断絶

- 先生はポーランド訛りの英語で投稿
- 保護者は日本語ネイティブ中心
- LINEアカウント名と本名が一致せず「誰が誰か」不明

#### 課題2: 情報の属人化

- 3教室の状況、生徒情報、発表会の進捗が全て先生の頭の中
- 統括保護者が質問に答えられない状態

#### 課題3: 発表会準備プロセスの不在

- 「演技の出来で直前に決めたい」という方針で演目・出演者が直前まで未確定
- 衣装・小道具・裏方タスクの準備は保護者側の責任
- 必要な情報が届かない

#### 課題4: 準備物の複雑性

- 衣装は演目や役によって異なる
- 入手方法も「購入」「手作り」「先生からの貸出」「保護者の手持ち」と多様
- 双方向の情報交換が必要

-----

## 3. 設計思想

### 3.1 柔軟性の確保

**固定化を避ける設計**

- 衣装・小道具・裏方タスクを統一的な「準備物・タスク」概念として扱う
- カテゴリやステータスは管理者が定義可能
- 関連付けは多対多で柔軟に設定

**変更への対応**

- 演目・出演者の直前変更に対応
- 準備物の追加・削除が容易
- ステータスのカスタマイズが可能

### 3.2 情報の可視化

**決定事項と未定事項の分離**

- 確定フラグによる明示的な状態管理
- 「TBD（未定）」状態の可視化
- 変更履歴の追跡

**ステータスの一元管理**

- 準備物の進捗を個別に追跡
- 未対応事項の自動検出
- ダッシュボードでの全体把握

### 3.3 運用負荷の最小化

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

**Cloudflare エッジコンピューティング**

|コンポーネント           |役割                    |
|------------------|----------------------|
|Cloudflare Workers|API・Webアプリケーションのホスティング|
|Cloudflare D1     |SQLiteベースのデータベース      |
|Workers AI        |多言語翻訳、コンテキスト解釈        |
|LINE Messaging API|更新通知のトリガー             |

### 4.2 アーキテクチャ

```
[ユーザー]
    ↓
[Cloudflare Workers]
    ↓
┌─────────────┬─────────────┬─────────────┐
│   D1 DB     │ Workers AI  │  LINE API   │
│ (データ管理) │  (翻訳)     │  (通知)     │
└─────────────┴─────────────┴─────────────┘
```

**データフロー**

1. 管理者が情報を入力/更新
1. 必要に応じて翻訳を生成（Workers AI）
1. 翻訳結果をDBに保存
1. 閲覧時はDBから取得（リアルタイム翻訳なし）
1. 更新時にLINE通知を送信

-----

## 5. 認証設計

### 5.1 パスキー認証（保護者向け）

**特徴**

- LINEアカウントとの紐づけ不要
- オフラインで完結する認証方式
- 形式: `ballet-sakura-7842` のような覚えやすい文字列

**運用フロー**

1. 管理者がシステムで発行
1. QRコード付きプリントに手書きで追記して配布
1. 保護者が初回アクセス時に入力
1. ブラウザに保存、以降は入力不要

**スコープ**

- 保護者単位（兄弟がいる場合は1つのパスキーで全員分を閲覧可能）

### 5.2 パスワード認証（管理者向け）

**役割と権限**

|役割       |権限                             |
|---------|-------------------------------|
|先生（owner）|全データの閲覧・編集、演目・出演者の確定、パスキー発行・無効化|
|サポート保護者  |情報の補足入力、パスキー発行、問い合わせ対応         |
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

**保護者 (Guardians)**

```sql
CREATE TABLE guardians (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  preferred_language TEXT NOT NULL, -- 'ja', 'en', 'ru'
  passkey TEXT UNIQUE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

**生徒 (Students)**

```sql
CREATE TABLE students (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  studio TEXT NOT NULL, -- 'A', 'B', 'C'
  guardian_id TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (guardian_id) REFERENCES guardians(id)
);
```

**発表会 (Recitals)**

```sql
CREATE TABLE recitals (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  date TEXT, -- ISO 8601
  venue TEXT,
  status TEXT NOT NULL, -- 'planning', 'preparation', 'completed'
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
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

#### 6.2.2 準備物・タスク管理

**準備物・タスク定義 (Items)**

衣装・小道具・裏方タスクを統一的に管理する柔軟なモデル。

```sql
CREATE TABLE items (
  id TEXT PRIMARY KEY,
  recital_id TEXT NOT NULL,
  type TEXT NOT NULL, -- 'costume', 'prop', 'backstage_task', 'other'
  category TEXT, -- 自由なカテゴリ（例: 'tutu', 'lighting', 'transportation'）
  name TEXT NOT NULL,
  description TEXT,
  attributes JSON, -- 柔軟な属性管理
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (recital_id) REFERENCES recitals(id)
);
```

**attributes の例**:

```json
{
  "acquisition_method": "purchase",
  "purchase_url": "https://...",
  "teacher_stock": 5,
  "making_instructions": "...",
  "deadline": "2024-12-20",
  "quantity_needed": 3
}
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
  updated_by TEXT NOT NULL, -- guardian_id or 'system'
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
  author_id TEXT NOT NULL, -- guardian_id
  content TEXT NOT NULL,
  is_question BOOLEAN DEFAULT FALSE,
  is_resolved BOOLEAN DEFAULT FALSE,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY (item_status_id) REFERENCES item_status(id)
);
```

#### 6.2.3 翻訳管理

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

#### 6.2.4 バレエ用語辞書

**用語集 (Glossary)**

翻訳精度向上のためのコンテキスト提供。

```sql
CREATE TABLE glossary (
  id TEXT PRIMARY KEY,
  term TEXT NOT NULL,
  category TEXT, -- 'technique', 'costume', 'music', etc.
  translation_ja TEXT,
  translation_en TEXT,
  translation_ru TEXT,
  notes TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  UNIQUE(term, category)
);
```

-----

## 7. 多言語対応戦略

### 7.1 翻訳対象の選別

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

### 7.2 翻訳生成のタイミング

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

### 7.3 TPOコンテキストの活用

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

### 7.4 データ容量の比較

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

## 8. 画面構成

### 8.1 管理者画面

#### ダッシュボード

- 発表会の進捗概要
- 未対応事項の一覧（未確認の準備物、未解決の質問など）
- 最近の更新履歴

#### 生徒・保護者管理

- 登録、編集、削除
- パスキー発行・無効化
- 生徒と保護者の紐づけ

#### 発表会管理

- 発表会の作成・編集
- 演目の登録、順番の変更
- キャスティング設定（演目×生徒×役名）
- 確定/未確定の切り替え

#### 準備物・タスク管理

- 準備物・タスクの作成（type, category, attributes を指定）
- 演目・役・生徒との関連付け
- ステータス定義のカスタマイズ
- 全体の進捗確認

#### コミュニケーション

- 保護者からのコメント・質問の確認
- 返信・解決済みマーク
- 問い合わせの多い項目の可視化

#### 翻訳管理

- 翻訳の再生成
- 用語辞書の編集
- 翻訳履歴の確認

#### 通知管理

- LINE通知の送信履歴
- 通知テンプレートの編集

### 8.2 保護者画面

#### マイページ

- 自分の子供に関する情報を一覧表示
- 発表会の日程・会場
- 出演する演目・役（確定/未確定）
- 準備物のチェックリスト

#### 発表会詳細

- 日程、会場、プログラム（全演目の一覧）
- 自分の子供が出演する演目をハイライト
- 確定/未確定の明示

#### 準備物一覧

- 衣装・小道具のチェックリスト
- ステータス更新（ドロップダウン選択）
- 入手方法の選択（購入/手作り/貸出/手持ち）
- コメント投稿（質問・相談）

#### 担当タスク

- 裏方タスクの確認
- ステータス更新
- コメント投稿

#### 言語切替

- 日本語 / 英語 / ロシア語
- ユーザー設定で希望言語を保存
- ページ内で即座に切り替え可能

-----

## 9. 通知設計

### 9.1 基本方針

**トリガーに徹する**

- LINE通知は「更新があった」ことを知らせるのみ
- 詳細情報はWebで参照する設計
- 通知頻度の制御（過度な通知を避ける）

### 9.2 通知パターン

#### 発表会情報の更新

```
発表会『春の舞』の情報が更新されました
→ [リンク]
```

#### 準備物の追加・変更

```
衣装の情報が追加されました。準備状況を確認してください
→ [リンク]
```

#### 質問への返信

```
あなたの質問に返信がありました
→ [リンク]
```

#### リマインダー

```
【至急】期日が近い準備物があります（12月20日まで）
→ [リンク]
```

### 9.3 通知の多言語化

保護者の希望言語に応じて通知を送信。

**日本語**:

```
発表会『春の舞』の情報が更新されました
→ https://ballet-system.example.com/recital/123
```

**英語**:

```
Recital "Spring Dance" information has been updated
→ https://ballet-system.example.com/recital/123
```

**ロシア語**:

```
Информация о концерте "Весенний танец" обновлена
→ https://ballet-system.example.com/recital/123
```

### 9.4 LINE Messaging API の利用

**送信先**:

- LINEグループチャット（全体通知）
- または個別通知（将来拡張）

**送信タイミング**:

- 管理者が手動で送信（通知管理画面から）
- 自動送信（期日リマインダーなど、将来拡張）

-----

## 10. 開発計画

### Phase 1: 基盤構築（1-2週間）

1. Cloudflare Workers + D1 のセットアップ
1. データベーススキーマの作成
1. 認証機能（パスキー・パスワード）の実装
1. 基本的な管理画面のUI構築

**成果物**:

- ログイン可能なシステム
- 生徒・保護者の登録機能

### Phase 2: コア機能（2-3週間）

1. 発表会・演目・キャスティング管理
1. 準備物・タスク管理とステータス追跡
1. 保護者向け画面の作成
1. 双方向コミュニケーション機能（コメント・質問）

**成果物**:

- 発表会の情報を管理・閲覧できるシステム
- 準備物のステータス追跡

### Phase 3: 連携と多言語（1-2週間）

1. LINE Messaging API 連携
1. Workers AI による翻訳機能
- TPOコンテキストの実装
- 用語辞書の整備
1. 裏方タスク管理

**成果物**:

- 多言語対応の完成
- LINE通知の送信

### Phase 4: 運用と改善（継続）

1. 次回発表会での試験運用
1. フィードバックに基づく改善
1. ドキュメント整備
1. ユーザーマニュアルの作成

**成果物**:

- 本番運用可能なシステム
- 運用マニュアル

-----

## 11. 技術スタック

|レイヤー   |技術                        |
|-------|--------------------------|
|ホスティング |Cloudflare Workers        |
|データベース |Cloudflare D1 (SQLite)    |
|AI     |Cloudflare Workers AI (翻訳)|
|フロントエンド|Hono + JSX または React      |
|外部連携   |LINE Messaging API        |
|認証     |パスキー（保護者）、パスワード（管理者）      |

-----

## 12. 運用コスト

Cloudflare の無料枠および低コストプランを活用。

|サービス              |コスト（概算）                    |
|------------------|---------------------------|
|Cloudflare Workers|無料枠で十分（10万リクエスト/日）         |
|Cloudflare D1     |無料枠で十分（5GB、500万行読み取り/日）    |
|Workers AI        |無料枠 + 従量課金（翻訳は編集時のみ、月数百円程度）|
|LINE Messaging API|無料（月200通まで、超過分は従量課金）       |
|**合計**            |**月額 0〜数百円**               |

**コスト削減のポイント**:

- リアルタイム翻訳を避け、編集時に事前生成
- 翻訳対象を選別（必要なものだけ翻訳）
- Cloudflare の無料枠を最大限活用

-----

## 13. 期待される効果

### 13.1 情報の可視化

- 「何が決まっているか分からない」問題の解消
- 情報が一元化され、最新状態が常に参照可能
- 決定事項と未定事項の明確な分離

### 13.2 言語の壁の解消

- 日本語・英語・ロシア語での情報アクセス
- 高精度な翻訳（TPOコンテキスト + 用語辞書）
- 全保護者が同等の情報にアクセス

### 13.3 準備物の抜け漏れ防止

- ステータス追跡による進捗の可視化
- 双方向コミュニケーションによる問題の早期発見
- 期日管理とリマインダー

### 13.4 統括者の負担軽減

- 「分からない」と答えるしかなかった状況から脱却
- システムを参照すれば答えられる状態へ
- 先生への質問集中を緩和

### 13.5 芸術的判断の尊重

- 直前までの変更に対応できる柔軟なシステム設計
- 確定/未確定の明示により、保護者も心の準備が可能
- 「完成度を妥協しない」方針を支援

-----

## 付録

### A. 用語集

|用語        |説明                                                 |
|----------|---------------------------------------------------|
|パスキー      |保護者向けの認証キー。`ballet-sakura-7842` のような形式             |
|TPO       |Time（時）、Place（場所）、Occasion（場合）の略。翻訳精度向上のためのコンテキスト情報|
|Workers AI|Cloudflare が提供するエッジAIサービス。翻訳などに利用                  |
|D1        |Cloudflare が提供するサーバーレスSQLiteデータベース                 |

### B. 参考資料

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare D1 Documentation](https://developers.cloudflare.com/d1/)
- [Cloudflare Workers AI Documentation](https://developers.cloudflare.com/workers-ai/)
- [LINE Messaging API Documentation](https://developers.line.biz/ja/docs/messaging-api/)

-----

**以上**