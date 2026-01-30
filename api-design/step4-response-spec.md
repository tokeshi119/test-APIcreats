# ステップ4: レスポンス構造のサンプル提示

## GET /api/v1/friends のレスポンス

### 成功時レスポンス（200 OK）

```json
{
  "total_count": 150,
  "page_number": 1,
  "limit": 100,
  "items": [
    {
      "id": "12345",
      "management_code": "U1234567890abcdef",
      "advertisement_code": "yokota_x",
      "line_id": "U1234567890abcdef",
      "registered_at": "2024-01-15T10:30:00+09:00",
      "status": "normal",
      "line_connection": {
        "line_official_account_id": "67890",
        "line_official_account_name": "LINE公式アカウント1"
      }
    },
    {
      "id": "12346",
      "management_code": "U9876543210fedcba",
      "advertisement_code": null,
      "line_id": "U9876543210fedcba",
      "registered_at": "2024-01-18T14:20:00+09:00",
      "status": "blocked",
      "line_connection": {
        "line_official_account_id": "67890",
        "line_official_account_name": "LINE公式アカウント1"
      }
    }
  ]
}
```

### レスポンスフィールド説明

#### トップレベル

| フィールド名 | 型 | 必須 | 説明 |
|------------|-----|------|------|
| `total_count` | integer | 必須 | 検索条件に該当する総件数 |
| `page_number` | integer | 必須 | 現在のページ番号（1始まり）<br>計算式: `floor(offset / limit) + 1` |
| `limit` | integer | 必須 | 1ページあたりの最大取得件数 |
| `items` | array | 必須 | 友だち情報の配列 |

#### items配列の各要素（友だち情報）

| フィールド名 | 型 | 必須 | 説明 | DDL対応 |
|------------|-----|------|------|---------|
| `id` | string | 必須 | 友だちID（管理ID） | `line_friends.id` |
| `management_code` | string | 必須 | 管理コード | `line_friends.messaging_api_user_id_cms` **【要確認】** 画面の「管理コード」列が実際に `messaging_api_user_id_cms` を表示しているか確認が必要 |
| `advertisement_code` | string \| null | 任意 | 広告コード | `advertisement_codes.advertisement_code` (JOIN) |
| `line_id` | string | 必須 | LINE ID | `line_friends.messaging_api_user_id` |
| `registered_at` | string | 必須 | 友だち登録日時（ISO 8601形式） | `line_friends.created_at` |
| `status` | string | 必須 | 友だち状態 | `line_friends.is_blocked` から変換<br>`normal`（正常）または `blocked`（ブロック） |
| `line_connection` | object | 必須 | LINE連携情報 | `line_friends.line_official_account_id` と関連情報 |

#### line_connection オブジェクト

| フィールド名 | 型 | 必須 | 説明 | DDL対応 |
|------------|-----|------|------|---------|
| `line_official_account_id` | string | 必須 | LINE公式アカウントID | `line_friends.line_official_account_id` |
| `line_official_account_name` | string | 任意 | LINE公式アカウント名 | `line_official_accounts` テーブルから取得（JOIN） |

### バックエンドへの裁量部分

以下はバックエンドで最適化可能です：

- **フィールド名**: スネークケース（`snake_case`）を推奨していますが、キャメルケース（`camelCase`）でも問題ありません
- **データ型**: `id` は数値型（`integer`）でも文字列型（`string`）でも構いませんが、文字列型を推奨（将来の拡張性のため）
- **日時形式**: ISO 8601形式を推奨しますが、他の形式でも問題ありません（ただし、タイムゾーン情報は含めることを推奨）
- **追加フィールド**: 必要に応じて追加フィールドを返しても問題ありません（例: `updated_at`, `query_string` など）
- **N+1問題回避**: `line_connection` オブジェクトの `line_official_account_name` は、JOINで取得するか、別APIで取得するかはバックエンドの判断に任せます
- **ページネーション情報**: `total_count` の計算が重い場合は、`has_next` や `has_prev` などのブール値で代替することも可能です

---

## DELETE /api/v1/friends/{friend_id} のレスポンス

### 成功時レスポンス（204 No Content）

レスポンスボディなし。HTTPステータスコードのみで成功を表現。

### エラーレスポンス（404 Not Found）

```json
{
  "type": "/types/404",
  "title": "Resource not found",
  "status": 404,
  "detail": "Friend with id '12345' not found"
}
```

### エラーレスポンス（403 Forbidden）

```json
{
  "type": "/types/403",
  "title": "Forbidden",
  "status": 403,
  "detail": "You do not have permission to delete this friend"
}
```

---

## 共通エラーレスポンス形式（RFC 9457準拠）

### 400 Bad Request（バリデーションエラー）

```json
{
  "type": "/types/400",
  "title": "Bad Request",
  "status": 400,
  "detail": "Invalid request parameters",
  "errors": [
    {
      "detail": "must be a positive integer",
      "pointer": "#/limit"
    },
    {
      "detail": "must be one of: normal, blocked",
      "pointer": "#/status"
    }
  ]
}
```

### 401 Unauthorized（認証エラー）

```json
{
  "type": "/types/401",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Authentication required"
}
```

### 500 Internal Server Error（サーバーエラー）

```json
{
  "type": "/types/500",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred"
}
```
