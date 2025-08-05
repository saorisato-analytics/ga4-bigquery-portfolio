# GA4 + BigQuery マーケティング分析基盤 ポートフォリオ

[![Portfolio Demo](https://img.shields.io/badge/Demo-Live%20Site-brightgreen)](https://saorisato-analytics.github.io/ga4-bigquery-portfolio/)
[![GA4](https://img.shields.io/badge/GA4-Tracking%20Active-blue)](https://analytics.google.com)
[![BigQuery](https://img.shields.io/badge/BigQuery-Connected-orange)](https://cloud.google.com/bigquery)
[![GCP](https://img.shields.io/badge/GCP-Powered-red)](https://cloud.google.com)

## 🎯 プロジェクト概要

このプロジェクトは、**GA4 + BigQuery + GCP**を活用したエンドツーエンドのマーケティングデータ分析基盤のデモンストレーションです。実際のEコマースサイトを模したデモサイトを通じて、リアルタイムデータ収集から高度な分析まで、実務レベルのデータエンジニアリング・分析スキルを展示しています。

### 🌟 主な特徴
- **完全動作するECサイト**：商品閲覧・カート・購入機能
- **リアルタイムデータ収集**：GA4による包括的なユーザー行動追跡
- **高度なSQL分析**：BigQueryでの複雑なデータ分析
- **スケーラブルアーキテクチャ**：GCPクラウドサービス連携
- **自動化されたパイプライン**：データ収集〜分析の完全自動化

## 🔗 Demo & Links

- **🌐 Live Demo Site**: [https://saorisato-analytics.github.io/ga4-bigquery-portfolio/](https://saorisato-analytics.github.io/ga4-bigquery-portfolio/)
- **📊 GA4 Property**: `Demo Ecommerce Site`
- **🗄️ BigQuery Dataset**: `analytics_*`
- **☁️ GCP Project**: `portfolio-analytics`

## 🛠️ 技術スタック

### Frontend
- **HTML5/CSS3/JavaScript**: モダンなレスポンシブデザイン
- **GitHub Pages**: 静的サイトホスティング

### Analytics & Data
- **Google Analytics 4 (GA4)**: 包括的なイベント追跡
- **Google Tag Manager**: タグ管理とデータレイヤー実装
- **BigQuery**: データウェアハウス・SQL分析
- **Google Cloud Platform**: クラウドインフラ

### Development
- **Git/GitHub**: バージョン管理・CI/CD
- **VS Code**: 開発環境

## 🏗️ システムアーキテクチャ

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Demo EC Site   │───▶│      GA4        │───▶│    BigQuery     │
│  (GitHub Pages)  │    │  (Event Track)  │    │ (Data Analysis) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│      GTM        │    │   Realtime      │    │  SQL Queries    │
│ (Tag Management)│    │   Dashboard     │    │   & Reports     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📊 実装済み分析機能

### イベント追跡
- **Page View**: ページ閲覧追跡
- **View Item**: 商品詳細閲覧
- **Add to Cart**: カート追加
- **Purchase**: 購入完了

### 分析クエリ
- **商品別パフォーマンス分析**: 閲覧・カート・購入の漏斗分析
- **ユーザー行動分析**: セッション・エンゲージメント分析
- **売上分析**: 日別・商品別売上レポート
- **コンバージョン分析**: CVR・ARPU計算

## 💡 技術的実装詳細

### GA4イベント実装例
```javascript
// カート追加イベント
gtag('event', 'add_to_cart', {
    currency: 'JPY',
    value: price,
    items: [{
        item_id: productId,
        item_name: productName,
        price: price,
        quantity: 1
    }]
});
```

### BigQuery分析クエリ例
```sql
-- 商品別パフォーマンス分析
SELECT 
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'item_id') as product_id,
  (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'item_name') as product_name,
  COUNT(CASE WHEN event_name = 'view_item' THEN 1 END) as views,
  COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END) as add_to_carts,
  COUNT(CASE WHEN event_name = 'purchase' THEN 1 END) as purchases,
  ROUND(COUNT(CASE WHEN event_name = 'add_to_cart' THEN 1 END) / 
        NULLIF(COUNT(CASE WHEN event_name = 'view_item' THEN 1 END), 0) * 100, 2) as cart_rate
FROM `portfolio-analytics.analytics_*.events_*`
WHERE event_name IN ('view_item', 'add_to_cart', 'purchase')
GROUP BY product_id, product_name
ORDER BY views DESC
```

## 📈 想定されるビジネス価値

### 💼 実務での活用シーン
- **Eコマース最適化**: 商品・UXの改善提案
- **マーケティング効果測定**: チャネル別ROI分析
- **在庫管理最適化**: 需要予測・仕入れ計画
- **顧客セグメンテーション**: パーソナライゼーション

### 📊 期待される成果
- **コンバージョン率向上**: 10-30%の改善
- **マーケティングROI改善**: 効果測定精度向上
- **運用工数削減**: 自動化による50%削減
- **意思決定スピード向上**: リアルタイム分析

## 🚀 今後の拡張予定

### Phase 2: 高度な分析機能
- [ ] **コホート分析**: ユーザーリテンション分析
- [ ] **RFM分析**: 顧客価値セグメンテーション
- [ ] **予測分析**: ML予測モデル実装
- [ ] **A/Bテスト基盤**: 統計的検定実装

### Phase 3: 自動化・可視化
- [ ] **Looker Studio ダッシュボード**: リアルタイム可視化
- [ ] **自動レポート生成**: 定期レポート配信
- [ ] **アラート機能**: 異常値検知・通知
- [ ] **API連携**: 外部システムとの連携

### Phase 4: スケールアップ
- [ ] **TreasureData連携**: マルチソースデータ統合
- [ ] **Claude API活用**: AI分析・インサイト生成
- [ ] **リアルタイムETL**: Dataflow実装
- [ ] **コスト最適化**: BigQuery BI Engine活用

## 📋 プロジェクト管理

### 開発プロセス
- **要件定義**: ビジネス要件から技術要件への落とし込み
- **設計書作成**: システム・データベース設計
- **実装**: アジャイル開発手法
- **テスト**: 単体・結合・運用テスト
- **デプロイ**: CI/CD自動化

### 品質管理
- **コードレビュー**: GitHub Pull Request
- **データ品質管理**: BigQuery Data Quality Rules
- **パフォーマンス監視**: GCP Monitoring
- **セキュリティ**: IAM・データ暗号化

## 🎯 スキル習得・証明項目

### データエンジニアリング
- ✅ **データパイプライン構築**: GA4 → BigQuery
- ✅ **ETL処理**: リアルタイム・バッチ処理
- ✅ **クラウドアーキテクチャ**: GCP活用
- ✅ **データ品質管理**: 監視・検証

### データ分析
- ✅ **SQL分析**: 複雑なクエリ・ウィンドウ関数
- ✅ **ファネル分析**: コンバージョン最適化
- ✅ **コホート分析**: ユーザー行動分析
- ✅ **統計分析**: A/Bテスト・仮説検定

### マーケティング分析
- ✅ **GA4実装**: 包括的な計測設計
- ✅ **GTM活用**: タグ管理・データレイヤー
- ✅ **KPI設計**: ビジネス指標の定義・追跡
- ✅ **レポーティング**: 経営層向け資料作成

## 📞 お問い合わせ

このポートフォリオについてご質問やお仕事のご相談がございましたら、お気軽にお声がけください。

### 得意領域
- **データ分析基盤構築**: GA4・BigQuery・GCP
- **マーケティング分析**: CVR改善・ROI最適化
- **自動化・効率化**: ETL・レポート自動化
- **要件定義**: ビジネス課題の技術的解決

---

## 📄 ライセンス

MIT License - 詳細は [LICENSE](LICENSE) ファイルをご確認ください。

---

**🎯 This portfolio demonstrates end-to-end data engineering and analytics capabilities using modern cloud technologies. Ready for immediate deployment in production environments.**
