# ステップ3: リクエスト仕様の設計

## GET /api/v1/friends

### クエリパラメータ

#### 検索・フィルタ関連

| パラメータ名 | 型 | 必須 | デフォルト値 | 説明 | 制約条件 |
|------------|-----|------|------------|------|---------|
| `q` | string | 任意 | - | 検索キーワード（管理コード、広告コード、LINE IDで部分一致検索） | 最大長: 255文字 |
| `status` | string | 任意 | - | 友だち状態でフィルタ | 列挙値: `normal`（正常）, `blocked`（ブロック） |
| `advertisement_code` | string | 任意 | - | 広告コードでフィルタ（完全一致） | - |
| `line_official_account_id` | integer | 任意 | - | LINE公式アカウントIDでフィルタ | 正の整数 |

#### ソート関連

| パラメータ名 | 型 | 必須 | デフォルト値 | 説明 | 制約条件 |
|------------|-----|------|------------|------|---------|
| `sort` | string | 任意 | `created_at:desc` | ソート条件 | 形式: `{field}:{order}`<br>field: `created_at`, `id`, `messaging_api_user_id_cms`<br>order: `asc`, `desc`<br>例: `created_at:desc`, `id:asc` |

#### ページング関連

| パラメータ名 | 型 | 必須 | デフォルト値 | 説明 | 制約条件 |
|------------|-----|------|------------|------|---------|
| `limit` | integer | 任意 | 100 | 1ページあたりの最大取得件数 | 最小値: 1<br>最大値: 100 |
| `offset` | integer | 任意 | 0 | 取得開始位置（オフセット） | 最小値: 0 |

### リクエスト例

```
GET /api/v1/friends?q=test&status=normal&sort=created_at:desc&limit=100&offset=0
```

### バックエンド確認事項

- [ ] `q` パラメータの検索対象フィールド（管理コード、広告コード、LINE IDの全てか、特定のフィールドのみか）
- [ ] 複数のフィルタ条件の組み合わせ（AND条件かOR条件か）
- [ ] ソート可能なフィールドの実装可否
- [ ] ページネーションのデフォルト値（100件で問題ないか）
- [ ] `offset` と `limit` の組み合わせで、最大取得件数の制限が必要か

---

## DELETE /api/v1/friends/{friend_id}

### パスパラメータ

| パラメータ名 | 型 | 必須 | 説明 | 制約条件 |
|------------|-----|------|------|---------|
| `friend_id` | string | 必須 | 友だちID（`line_friends.id` の値） | 正の整数を文字列として表現 |

### リクエスト例

```
DELETE /api/v1/friends/12345
```

### バックエンド確認事項

- [ ] 削除処理の実装方法（論理削除か物理削除か）
- [ ] 論理削除の場合、削除フラグのフィールド名
- [ ] 削除済みリソースの削除リクエスト時の挙動（204 No Contentを返すか、404 Not Foundを返すか）
- [ ] 削除権限のチェック（特定の権限が必要か）

---

## 共通リクエストヘッダ

| ヘッダ名 | 型 | 必須 | 説明 |
|---------|-----|------|------|
| `Authorization` | string | 必須 | Bearer Token形式の認証トークン<br>形式: `Bearer {access_token}` |
| `Content-Type` | string | 任意 | `application/json`（GET/DELETEでは通常不要） |
| `Accept` | string | 任意 | `application/json`（デフォルト） |

### リクエストヘッダ例

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```
