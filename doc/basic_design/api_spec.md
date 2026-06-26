# API仕様

## 概要

Electric Sheep のバックエンド API は、Web / Mobile Client から利用される REST API とする。入出力形式は JSON を基本とし、メディアアップロードのみ `multipart/form-data` を利用する。

ベースパスは `/v1` とする。ヘルスチェックのみ運用監視用として `/health` を提供する。

## 認証

認証方式は JWT Bearer Token とする。認証が必要なAPIでは、クライアントは以下のヘッダを付与する。

```http
Authorization: Bearer <access_token>
```

ログイン後に `access_token` と `refresh_token` を返す。

- `access_token` は短期有効とし、通常APIの認証に利用する。
- `refresh_token` は長期有効とし、アクセストークン再発行に利用する。
- `refresh_token` は平文では保存せず、ハッシュ化してDBに保存する。
- ログアウト時は該当する `refresh_token` を失効状態に更新する。

## 共通レスポンス

### 成功レスポンス

```json
{
  "data": {}
}
```

一覧取得ではページング情報を含める。

```json
{
  "data": [],
  "pagination": {
    "limit": 20,
    "offset": 0,
    "total": 100
  }
}
```

### エラーレスポンス

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "invalid request",
    "details": {}
  }
}
```

## エンドポイント一覧

### Health

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| GET | `/health` | 不要 | ヘルスチェック |

レスポンス:

```json
{
  "status": "ok"
}
```

### Auth

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| POST | `/v1/auth/register` | 不要 | ユーザー登録 |
| POST | `/v1/auth/login` | 不要 | ログイン |
| POST | `/v1/auth/logout` | 必要 | ログアウト |
| POST | `/v1/auth/refresh` | 不要 | アクセストークン再発行 |
| GET | `/v1/me` | 必要 | 自分のユーザー情報取得 |

パスワードリセットは初期MVPでは提供しない。

#### POST /v1/auth/register

リクエスト:

```json
{
  "name": "string",
  "email": "user@example.com",
  "password": "string"
}
```

レスポンス: `201 Created`

```json
{
  "data": {
    "id": "uuid",
    "name": "string",
    "email": "user@example.com"
  }
}
```

#### POST /v1/auth/login

リクエスト:

```json
{
  "email": "user@example.com",
  "password": "string"
}
```

レスポンス:

```json
{
  "data": {
    "access_token": "string",
    "refresh_token": "string",
    "token_type": "Bearer",
    "expires_in": 3600
  }
}
```

### Memories

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| POST | `/v1/memories` | 必要 | 思い出投稿を作成 |
| GET | `/v1/memories` | 必要 | 自分の投稿一覧を取得 |
| GET | `/v1/memories/{memory_id}` | 必要 | 投稿詳細を取得 |
| GET | `/v1/memories/today` | 必要 | 過去の同日投稿を取得 |
| GET | `/v1/memories/random` | 必要 | ランダムな過去投稿を取得 |

#### POST /v1/memories

`multipart/form-data` で本文とメディアを送信する。

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `occurred_at` | datetime | 必須 | 出来事が起きた日時 |
| `title` | string | 任意 | 投稿タイトル |
| `body` | string | 必須 | 思い出本文 |
| `emotion` | string | 任意 | `happy`, `fun`, `neutral`, `sad`, `angry` など |
| `tags` | string[] | 任意 | タグ配列 |
| `visibility` | string | 必須 | `private`, `mutual_followers` |
| `media` | file[] | 任意 | 写真、動画、音声 |

レスポンス: `201 Created`

```json
{
  "data": {
    "id": "uuid",
    "occurred_at": "2026-06-26T12:00:00+09:00",
    "title": "string",
    "body": "string",
    "emotion": "happy",
    "tags": ["school", "family"],
    "visibility": "private",
    "media": [
      {
        "id": "uuid",
        "media_type": "image",
        "url": "https://example.com/media/..."
      }
    ],
    "created_at": "2026-06-26T12:00:00+09:00"
  }
}
```

#### GET /v1/memories

クエリパラメータ:

| パラメータ | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `limit` | number | 任意 | 取得件数。デフォルト20 |
| `offset` | number | 任意 | 取得開始位置。デフォルト0 |
| `tag` | string | 任意 | タグ絞り込み |
| `emotion` | string | 任意 | 感情絞り込み |
| `from` | date | 任意 | 期間開始 |
| `to` | date | 任意 | 期間終了 |

### Tags

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| GET | `/v1/tags` | 必要 | 自分が使ったタグ一覧を取得 |

### Stats

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| GET | `/v1/stats/summary` | 必要 | 総投稿数、連続投稿日数などを取得 |
| GET | `/v1/stats/emotions` | 必要 | 感情ログ集計を取得 |

#### GET /v1/stats/summary

レスポンス:

```json
{
  "data": {
    "total_memories": 123,
    "current_streak_days": 7,
    "longest_streak_days": 30
  }
}
```

### Follow / Share

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| POST | `/v1/follows/{user_id}` | 必要 | フォロー申請 |
| POST | `/v1/follows/{user_id}/accept` | 必要 | フォロー申請承認 |
| POST | `/v1/follows/{user_id}/reject` | 必要 | フォロー申請拒否 |
| DELETE | `/v1/follows/{user_id}` | 必要 | フォロー解除 |
| GET | `/v1/follows` | 必要 | フォロー一覧を取得 |
| GET | `/v1/shared-memories` | 必要 | 相互フォロワーから共有された投稿を取得 |

フォローは申請制とする。`visibility = mutual_followers` の投稿は、双方のフォロー関係が承認済みの場合のみ閲覧できる。

### Search / AI

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| POST | `/v1/search` | 必要 | 自分の投稿を自然文で検索 |
| POST | `/v1/memories/{memory_id}/summary` | 必要 | 投稿の要約を生成 |

AI検索・要約の対象は自分の投稿のみとする。検索・要約では本文、タグ、感情、写真、動画、音声を入力候補に含める。共有された投稿は対象外とする。生成AIプロバイダはMVPでは Gemini API とし、モデルは Gemini 2.5 Flash を第一候補とする。

#### POST /v1/search

リクエスト:

```json
{
  "query": "家族で海に行った楽しい思い出",
  "limit": 10
}
```

レスポンス:

```json
{
  "data": {
    "query": "家族で海に行った楽しい思い出",
    "results": [
      {
        "memory_id": "uuid",
        "score": 0.92,
        "snippet": "string"
      }
    ]
  }
}
```

### Export

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| POST | `/v1/exports` | 必要 | 投稿データとメディアを含むエクスポートを作成 |
| GET | `/v1/exports/{export_id}` | 必要 | エクスポート状態とダウンロードURLを取得 |

エクスポート対象は自分の投稿のみとし、共有された他ユーザーの投稿は含めない。エクスポート形式はZIPとし、以下の構成を基本とする。

```text
export.zip
  manifest.json
  media/
    <memory_id>/
      <media_id>.<extension>
```

`manifest.json` には投稿本文、タグ、感情、公開範囲、メディアの相対パス、MIMEタイプ、ファイルサイズ、チェックサムを含める。外部ストレージの公開URLには依存せず、ZIP単体で別媒体へ移行できる状態にする。

### Notifications

| メソッド | パス | 認証 | 概要 |
| --- | --- | --- | --- |
| GET | `/v1/notifications` | 必要 | アプリ内通知一覧を取得 |
| POST | `/v1/notifications/{notification_id}/read` | 必要 | 通知を既読にする |

通知方式はMVPではアプリ内通知のみとする。メール通知とPush通知は提供しない。

## ステータスコード

| ステータス | 用途 |
| --- | --- |
| 200 | 取得・更新成功 |
| 201 | 作成成功 |
| 204 | レスポンスボディなしの成功 |
| 400 | リクエスト形式不正 |
| 401 | 未認証 |
| 403 | 権限不足 |
| 404 | 対象リソースなし |
| 409 | 一意制約違反などの競合 |
| 422 | バリデーションエラー |
| 429 | レート制限 |
| 500 | サーバー内部エラー |

## バージョニング方針

- API は URL ベースで `/v1` のようにバージョンを持つ。
- 破壊的変更を行う場合は `/v2` を追加する。
- 同一バージョン内では後方互換を維持する。
