# Sarashina2-Vision-API 要件定義・基本仕様

## 1. 目的
HuggingFaceのsarashina2-visionシリーズ（8B/14B）を利用し、OpenAI互換API（主に `/v1/chat/completions`）を提供する。

## 2. 主な要件
- OpenAI API互換（リクエスト/レスポンス形式・エンドポイント）
- 画像入力をメインとしたマルチモーダル推論
- ストリーミングレスポンス（OpenAIのstream: true対応）
- 認証: 本APIサーバーの責務外とし、API Gatewayやリバースプロキシ等の上位レイヤーで対応する想定。
- デプロイはDockerコンテナで行う
- 推論はGPU必須
- 外部公開も想定（セキュリティ考慮）

## 3. モデル
- 利用モデルは `sbintuitions/sarashina2-vision-8b` のみ（他モデルはサポートしない）

## 4. 基本仕様

### エンドポイント
- `POST /v1/chat/completions`
- `GET /v1/models`

`/v1/models` はOpenAI互換のモデル一覧取得エンドポイントであり、利用可能なモデル（`sbintuitions/sarashina2-vision-8b`のみ）を返します。

### 認証
- 本APIサーバー自体では認証処理を行いません。
- 認証は、APIゲートウェイやリバースプロキシなど、上位のレイヤーで実施されることを想定します。

### リクエスト例
```json
{
  "model": "sbintuitions/sarashina2-vision-8b",
  "messages": [
    {
      "role": "user",
      "content": [
        {"type": "text", "text": "この画像について説明してください"},
        {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}
      ]
    }
  ],
  "max_tokens": 128,
  "stream": true
}
```
- "model" は "sbintuitions/sarashina2-vision-8b" のみ指定可能

### レスポンス（stream: true時）
- OpenAI形式で1行ごとに `data: {JSON}` で返却
- 最後に `data: [DONE]`
- エラーレスポンスもOpenAI API互換形式

### 画像入力
- "content" 配列の中で type: "image_url" を指定し、base64エンコード画像を "url" にセット
- 対応フォーマットや画像サイズ等は sarashina2-vision-8b がサポートするものに準拠（詳細はHuggingFace公式モデルカード参照）

### モデル切り替え
- モデルの切り替えはサポートせず、`sbintuitions/sarashina2-vision-8b`のみ利用可能

## 5. 備考
- 画像とテキストの組み合わせによるマルチモーダル推論が可能
- DockerコンテナはCUDA対応ベースイメージを推奨
- モデルは事前にダウンロードし、ボリュームマウントで永続化することで再起動時も高速化・ネットワーク節約
- ログはDocker/Kubernetes標準（stdout/stderr）で出力。永続化や集約は基盤側で対応
- 利用制限やレートリミットはAPI Gatewayやリバースプロキシ等で対応する想定（本APIでは未実装）
- CORSは一般的な設定（全許可またはオリジン指定可）。必要に応じて後から調整可能
- HTTPSはリバースプロキシ側で対応し、アプリ本体はHTTPのみでOK
