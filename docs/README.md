# 教室運営支援システム 設計ドキュメント

**作成日**: 2024年12月
**更新日**: 2025年12月
**バージョン**: 3.0

---

## 概要

習い事教室における**日々のレッスン運営**と**発表会・イベント運営**の課題を解決する多言語対応マルチテナントWebシステムの設計ドキュメントです。

## ドキュメント構成

### アーキテクチャ

- **[システム概要と設計思想](./architecture/overview.md)**
  システムの概要、コアバリュー、主要機能、および設計思想について

- **[マルチテナントアーキテクチャ](./architecture/multi-tenant.md)**
  シングルバイナリ・マルチテナント設計、テナント識別、データ分離戦略

- **[システム構成と技術スタック](./architecture/system-architecture.md)**
  技術基盤、LINE Platform構成、アーキテクチャ、データフロー、技術スタック

### 設計

- **[認証設計](./design/authentication.md)**
  LINE認証（保護者向け）、パスワード認証（管理者向け）

- **[データモデル設計](./design/data-model.md)**
  設計原則、主要エンティティ、テーブル定義

- **[セキュリティ設計](./design/security.md)**
  設計原則、役割と権限、プライバシー保護、データアクセス制御

- **[多言語対応戦略](./design/i18n.md)**
  翻訳対象の選別、翻訳生成タイミング、TPOコンテキスト活用

- **[通知設計](./design/notification.md)**
  プル型を基本とした通知戦略、Reply Token応答、手動プッシュ通知

### UI

- **[画面構成](./ui/screens.md)**
  管理者画面、保護者画面（LIFFアプリ）、LINEメッセージ応答

### プロジェクト

- **[開発計画](./project/development-plan.md)**
  Phase 1〜5の開発ステップ

- **[運用コスト](./project/cost.md)**
  Cloudflare と LINE Platform の運用コスト試算

- **[設計課題](./project/issues.md)**
  実装に向けて詳細設計が必要な課題の整理

- **[期待される効果](./project/benefits.md)**
  システム導入により期待される効果

### テナント固有設定

- **[子供バレエ教室](./tenants/ballet-school.md)**
  バレエ教室テナントの固有設定（教室概要、利用者特性、運営パターン、準備物カテゴリ、ステータス定義、マイルストーンテンプレート、通知テンプレート、バレエ用語辞書、LINE公式アカウント設定など）

---

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

---

**以上**
