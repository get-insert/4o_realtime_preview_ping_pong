# 4o_realtime_preview_ping_pong

AzureやOpenAIのRealtimeAPI（4o-realtime-preview）のレイテンシーや安定性が時間帯や時期によって安定しないという疑惑をはっきりさせるためのAPI監視サービス

## 📋 プロジェクト概要

このプロジェクトは、OpenAI Realtime API（4o-realtime-preview）とAzure Speech Servicesのリアルタイム音声処理機能の性能と安定性を継続的に監視・評価するためのツールです。

### 🎯 主な目的

- **レイテンシー測定**: セッション開始から音声認識・合成までの各段階のレイテンシーを測定
- **安定性評価**: 時間帯や時期による性能変動の監視
- **WebSocket接続性能**: 接続確立からセッション開始までの時間測定
- **ファンクションコール性能**: リアルタイムファンクションコールの応答時間と成功率の監視

## 🔧 技術仕様

### 監視対象API
- **OpenAI Realtime API (4o-realtime-preview)**
  - WebSocket接続によるリアルタイム音声処理
  - 音声認識（Speech-to-Text）
  - 音声合成（Text-to-Speech）
  - リアルタイムファンクションコール

- **Azure Speech Services**
  - 同等のリアルタイム音声処理機能
  - 比較対象としての性能評価

## 📡 OpenAI Realtime API 詳細仕様

### WebSocket接続仕様
- **エンドポイント**: `wss://api.openai.com/v1/audio/realtime`
- **認証**: Bearer Token（API Key）
- **プロトコル**: WebSocket over TLS
- **接続タイムアウト**: 30秒
- **メッセージ形式**: JSON

### クライアント→サーバーイベント（送信）

#### 1. セッション開始
```json
{
  "type": "session.start",
  "data": {
    "model": "gpt-4o-realtime-preview",
    "voice": "alloy",
    "input_audio_format": "pcm_s16le",
    "output_audio_format": "pcm_s16le",
    "sample_rate": 24000,
    "chunk_length_s": 0.1
  }
}
```

#### 2. 音声データ送信
```json
{
  "type": "input_audio",
  "data": {
    "audio": "base64_encoded_audio_data"
  }
}
```

#### 3. テキスト入力
```json
{
  "type": "input_text",
  "data": {
    "text": "Hello, how are you?"
  }
}
```

#### 4. ファンクションコール
```json
{
  "type": "function_call",
  "data": {
    "name": "get_weather",
    "arguments": "{\"location\": \"Tokyo\"}"
  }
}
```

#### 5. セッション終了
```json
{
  "type": "session.end"
}
```

### サーバー→クライアントイベント（受信）

#### 1. セッション開始確認
```json
{
  "type": "session.started",
  "data": {
    "session_id": "session_123"
  }
}
```

#### 2. 音声認識結果
```json
{
  "type": "speech.started",
  "data": {
    "timestamp": 1234567890
  }
}
```

```json
{
  "type": "speech.partial",
  "data": {
    "text": "Hello, how",
    "timestamp": 1234567890
  }
}
```

```json
{
  "type": "speech.final",
  "data": {
    "text": "Hello, how are you?",
    "timestamp": 1234567890
  }
}
```

#### 3. 音声合成結果
```json
{
  "type": "audio.started",
  "data": {
    "timestamp": 1234567890
  }
}
```

```json
{
  "type": "audio",
  "data": {
    "audio": "base64_encoded_audio_data",
    "timestamp": 1234567890
  }
}
```

```json
{
  "type": "audio.finished",
  "data": {
    "timestamp": 1234567890
  }
}
```

#### 4. ファンクションコール結果
```json
{
  "type": "function_result",
  "data": {
    "name": "get_weather",
    "result": "{\"temperature\": 25, \"condition\": \"sunny\"}",
    "timestamp": 1234567890
  }
}
```

#### 5. エラーイベント
```json
{
  "type": "error",
  "data": {
    "error": {
      "type": "invalid_request",
      "message": "Invalid audio format"
    }
  }
}
```

## ⏱️ 詳細測定項目

### 1. 接続・セッション開始レイテンシー
- **WebSocket接続確立時間**: 接続開始から`onopen`イベントまで
- **認証処理時間**: 接続確立から認証完了まで
- **セッション初期化時間**: `session.start`送信から`session.started`受信まで
- **音声処理準備時間**: セッション開始から最初の音声処理可能状態まで

### 2. 音声認識レイテンシー（Speech-to-Text）
- **音声入力開始時間**: 最初の音声データ送信から`speech.started`受信まで
- **部分認識レイテンシー**: 音声データ送信から`speech.partial`受信まで
- **最終認識レイテンシー**: 音声データ送信から`speech.final`受信まで
- **認識精度**: 部分認識と最終認識のテキスト一致度

### 3. 音声合成レイテンシー（Text-to-Speech）
- **テキスト入力時間**: `input_text`送信から`audio.started`受信まで
- **音声生成時間**: `audio.started`から最初の`audio`チャンク受信まで
- **音声完了時間**: `audio.started`から`audio.finished`受信まで
- **音声品質**: 生成された音声の長さと品質評価

### 4. ファンクションコールレイテンシー
- **ファンクション実行時間**: `function_call`送信から`function_result`受信まで
- **ファンクション成功率**: 成功したファンクションコールの割合
- **エラーハンドリング時間**: エラー発生から`error`イベント受信まで

### 5. リアルタイム性能指標
- **音声チャンク間隔**: 連続する音声データチャンク間の時間間隔
- **イベント処理時間**: 各イベントの処理にかかる時間
- **バッファリング状況**: 音声バッファの充填率と遅延状況
- **接続安定性**: 接続断線・再接続の頻度と時間

### 6. サーバー側イベントタイミング
- **イベント順序の整合性**: 期待されるイベント順序との一致度
- **タイムスタンプ精度**: サーバー側タイムスタンプの一貫性
- **イベント間隔**: 連続するイベント間の時間間隔
- **同時処理能力**: 複数の音声・テキスト処理の並行性

### 7. エラー・異常検知
- **接続エラー率**: WebSocket接続失敗の頻度
- **認証エラー率**: API認証失敗の頻度
- **処理エラー率**: 音声・テキスト処理エラーの頻度
- **タイムアウト発生率**: 各処理のタイムアウト発生頻度

## 📊 監視スケジュール

- **定期監視**: 1分間隔での自動ピンポンテスト
- **詳細分析**: 時間帯別（朝・昼・夕・夜）の性能比較
- **長期トレンド**: 日次・週次・月次の性能推移分析
- **アラート機能**: 閾値を超えた性能劣化時の通知

## 🚀 実装予定機能

### コア機能
- [ ] WebSocket接続管理
- [ ] 音声データの自動生成・送信
- [ ] レイテンシー測定ロジック
- [ ] データ収集・保存システム
- [ ] リアルタイムダッシュボード

### 分析機能
- [ ] 統計分析（平均、分散、パーセンタイル）
- [ ] 時系列分析
- [ ] 異常検知アルゴリズム
- [ ] レポート生成機能

### 運用機能
- [ ] アラート・通知システム
- [ ] ログ管理
- [ ] 設定管理UI
- [ ] API認証情報管理

## 🛠 技術スタック（AWSサーバーレス構成）

### フロントエンド
- **React** / **TypeScript** - SPAアプリケーション
- **Chart.js** / **D3.js** - データ可視化
- **Tailwind CSS** - UIスタイリング
- **AWS S3** - 静的ホスティング
- **CloudFront** - CDN配信

### バックエンド（サーバーレス）
- **AWS Lambda** - サーバーレス関数
  - WebSocket接続管理
  - レイテンシー測定ロジック
  - データ処理・分析
- **API Gateway** - REST API・WebSocket API
- **DynamoDB** - NoSQLデータベース
  - 時系列データ保存
  - 設定情報管理
  - ユーザーセッション管理

### インフラ・運用
- **AWS CloudWatch** - ログ・メトリクス監視
- **AWS EventBridge** - スケジュール実行
- **AWS SNS** - アラート通知
- **GitHub Actions** - CI/CD
- **AWS SAM** - サーバーレスアプリケーション管理

### コスト最適化
- **従量課金**: 使用した分だけの課金
- **自動スケーリング**: 負荷に応じた自動リソース調整
- **無料枠活用**: AWS Lambda、DynamoDBの無料枠を最大活用
- **データ保持期間**: 古いデータの自動削除によるストレージコスト削減

## 📈 期待される成果

1. **定量的な性能評価**
   - 各APIの実際の性能データの蓄積
   - 時間帯による性能変動の可視化
   - ベンチマーク比較の実現

2. **運用改善**
   - 性能劣化の早期発見
   - 最適な時間帯の特定
   - コスト効率の最適化

3. **技術的知見**
   - リアルタイムAPIの性能特性の理解
   - 最適な実装パターンの確立
   - 障害対策の強化

## 🔄 開発フェーズ

### Phase 1: 基本実装
- AWS Lambda + DynamoDBの基本構成
- WebSocket接続の基本実装
- 単純なピンポンテスト
- 基本的なレイテンシー測定

### Phase 2: 機能拡張
- 音声処理の統合
- ファンクションコールテスト
- S3静的ホスティングの設定
- 基本的なダッシュボード

### Phase 3: 分析・可視化
- 詳細なダッシュボード実装
- 統計分析機能
- アラート機能
- CloudWatch統合

### Phase 4: 運用・最適化
- 本格運用開始
- コスト最適化
- 機能拡張
- パフォーマンスチューニング

## ☕ コーヒー支援

このプロジェクトの開発・運用には継続的なコストがかかります。もしこのプロジェクトが役に立ったら、コーヒー一杯分の支援をいただけますと幸いです。

- **Buy Me a Coffee**: [準備中]
- **Ko-fi**: [準備中]
- **GitHub Sponsors**: [準備中]

ご支援いただいた方には、特別な機能へのアクセスや優先サポートを提供予定です。

## 📝 ライセンス

MIT License

## 🤝 コントリビューション

このプロジェクトへの貢献を歓迎します。以下の方法で参加できます：

- バグレポート
- 機能要望
- プルリクエスト
- ドキュメント改善

## 📞 連絡先

プロジェクトに関する質問や提案がありましたら、Issueを作成してください。

