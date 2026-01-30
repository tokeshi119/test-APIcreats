# ステップ2: エンドポイント設計

## RESTfulリソースの特定

### 主要リソース
- **friends**: 友だちリソース（複数形）
- リソース名は `friends` を使用（`line-friends` ではなく、シンプルに）

### リソース間の関係
- `friends` は独立したリソース
- `advertisement_codes` や `line_official_accounts` は関連情報として含める（ネストしない）

## エンドポイント一覧

| メソッド | パス | 用途 | 備考 |
|---------|------|------|------|
| GET | `/api/v1/friends` | 友だち一覧取得 | ページネーション、検索、フィルタ、ソート対応 |
| DELETE | `/api/v1/friends/{friend_id}` | 友だち削除 | 特定の友だちを削除 |

## HTTPメソッドの選定理由

### GET /api/v1/friends
- **理由**: リソースの参照操作
- **冪等性**: あり
- **副作用**: なし（ログ記録などは除く）

### DELETE /api/v1/friends/{friend_id}
- **理由**: リソースの削除操作
- **冪等性**: あり（削除済みリソースの削除も成功として扱う）
- **副作用**: なし（削除操作自体は副作用だが、HTTPの意味での副作用はなし）

## パスパラメータ

### {friend_id}
- **型**: string
- **説明**: 友だちを識別するID
- **値の候補**:
  - `line_friends.id` (BIGINT) - 主キー
  - `line_friends.messaging_api_user_id` (TEXT) - LINE ID
- **推奨**: `line_friends.id` を使用（数値型だが、APIでは文字列として扱う）

## バージョニング

- **方式**: パス方式（`/api/v1/`）
- **理由**: URLに明示的で、クライアントがバージョンを指定しやすい
- **現在のバージョン**: v1

## カスタムメソッド

現時点では不要。将来的に以下のような操作が必要になる可能性:
- `POST /api/v1/friends/{friend_id}/block` - ブロック操作
- `POST /api/v1/friends/{friend_id}/unblock` - ブロック解除操作
- `POST /api/v1/friends/batch-delete` - 一括削除

これらは画面要件に含まれていないため、現時点では設計対象外。
