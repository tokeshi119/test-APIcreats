# API設計ガイド（詳細）

## 目次
1. 基本原則
2. HTTPメソッドの使い分け
3. リソース設計
4. APIバージョニング
5. リクエスト設計
6. レスポンス設計
7. ページング
8. 認証・認可
9. 楽観ロック
10. ファイル連携
11. バッチAPI
12. 設計時のチェックリスト
13. 参考資料

## 基本原則

### REST志向の設計

- **リソース中心**: URLはリソースを表現し、HTTPメソッドで操作を表現する
- **標準HTTPメソッドの使用**: GET（参照）、POST（作成）、PUT（置換）、PATCH（部分更新）、DELETE（削除）
- **ステートレス**: 各リクエストは独立しており、サーバー側でセッション状態を保持しない
- **統一されたインターフェース**: 一貫したURL構造、エラーハンドリング、レスポンス形式

### 命名規則

- **パス（リソース名）**: kebab-case + 複数形（例: `delivery-schedules`）
- **クエリパラメータ**: snake_case（例: `order_id`）
- **リクエスト/レスポンスボディのJSON項目**: snake_case（例: `order_id`）
- **リクエスト/レスポンスヘッダ**: kebab-case（例: `x-debug-enabled`）

## HTTPメソッドの使い分け

| メソッド | 用途 | 冪等性 | 副作用 |
|---------|------|--------|--------|
| GET | リソースの参照 | あり | あり |
| POST | リソースの作成、複雑な検索 | なし | なし |
| PUT | リソースの置換 | あり | なし |
| PATCH | リソースの部分更新 | 条件付き | なし |
| DELETE | リソースの削除 | あり | なし |

### 重要な注意点

- **新規作成はPOST**: PUTでリソースを新規作成するのは非推奨（理由があれば許容）
- **更新はPUTを優先**: PATCHは実装が複雑になるため、PUTで統一することを推奨
- **複雑な検索はPOST**: クエリパラメータで表現が難しい場合、リクエストボディでPOSTを使用

## リソース設計

### ネスト vs フラット

**ネスト表現** (`/articles/1/comments/1`) を採用する場合:
- 親リソースに紐づいた子リソースを一覧検索で取得する可能性がある
- 親リソースの配下にリソースを作成する
- 親リソースが削除された場合に、同時に子リソースも削除すべき

**フラット表現** (`/comments/1`) を採用する場合:
- 他のリソースと親子関係にない、独立したリソースである
- 親リソースが削除されても子リソースは残すべき

### カスタムメソッド

HTTPメソッドで表現できない操作（`copy`, `move`, `cancel`など）は、パス表現を使用:
- 例: `POST /drafts/{draft_id}/copy`
- カスタムメソッド部分はcamelCaseで記載
- カスタムメソッドはURLの最後の要素でのみ利用可能
- カスタムメソッドで表現する場合、`POST`メソッドを利用

## APIバージョニング

### バージョニング方式

- **パス方式を採用**: `/v1/users`, `/v2/users`
- メジャーバージョン粒度で管理（マイナーバージョン粒度は非推奨）
- 後方互換性を保った改修は同じバージョン内で対応
- 後方互換性を破壊する改修はメジャーバージョンを上げる

### 後方互換性

**後方互換性を保つ改修例**:
- 新規に任意属性のクエリパラメータを追加
- レスポンスボディのJSON項目を追加

**後方互換性を破壊する改修例**:
- リソースパスの変更
- クエリパラメータの廃止
- レスポンスからJSON項目を削除または名称変更
- APIの振る舞いを変更（同期→非同期など）

## リクエスト設計

### リクエストヘッダ

必須/推奨ヘッダ:
- `Accept: application/vnd.github+json` または `application/json`
- `X-GitHub-Api-Version` または `X-API-Version`: APIバージョン指定
- `Authorization`: 認証トークン（Bearer形式）
- `Content-Type: application/json`
- `User-Agent`: クライアント識別用

### クエリパラメータ

標準的なクエリパラメータ:
- `q`: 検索用キーワード
- `sort`: ソート条件（例: `sort=publish_status:asc,release_at:desc`）
- `fields`: 取得フィールドの絞り込み
- `limit`: 最大取得件数
- `page`: ページ番号（ページング時）

**注意**: GET/HEAD以外のメソッドでのクエリパラメータは原則禁止（DELETEのlock_noのみ例外）

### リクエストボディ

- JSON形式で送信
- `null`は原則使用せず、キー自体を含めない（`undefined`）で表現
- PATCHの場合はJSON Merge Patch形式（RFC 7386）を使用

## レスポンス設計

### HTTPステータスコード

| コード | 用途 | 使用例 |
|--------|------|--------|
| 200 OK | 検索成功、更新成功 | GET, PUT, PATCH |
| 201 Created | 登録処理で正常終了（同期） | POST |
| 202 Accepted | 非同期処理の呼び出しで正常終了 | POST |
| 204 No Content | 正常終了かつ、空で応答 | DELETE |
| 400 Bad Request | 入力バリデーションエラー | 全メソッド |
| 401 Unauthorized | 認証の失敗 | 全メソッド |
| 403 Forbidden | 認可の失敗（権限エラー） | 全メソッド |
| 404 Not Found | 存在しないリソースを指定 | 全メソッド |
| 409 Conflict | 楽観ロックエラー、一意制約違反 | PUT, PATCH |
| 422 Unprocessable Entity | 入力値が業務処理を行う条件を満たさない | 全メソッド |
| 429 Too Many Requests | レートリミット超過 | 全メソッド |
| 500 Internal Server Error | システムエラー | 全メソッド |

### レスポンスボディ

**一覧検索の形式**:
```json
{
  "total_count": 2,
  "page_number": 1,
  "items": [
    { "id": 1, "name": "Item 1" },
    { "id": 2, "name": "Item 2" }
  ]
}
```

**エラーレスポンス** (RFC 9457準拠):
```json
{
  "type": "/types/123",
  "title": "Your request is not valid.",
  "errors": [
    {
      "detail": "must be a positive integer",
      "pointer": "#/age"
    }
  ]
}
```

### レスポンスヘッダ

- `Content-Type: application/json` または `application/problem+json`（エラー時）
- `Cache-Control: no-store`（業務システムの場合、キャッシュを禁止）
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`: レートリミット情報

## ページング

### オフセット&リミット方式（RDBの場合）

```
GET /users?limit=10&offset=20
```

レスポンス:
```json
{
  "total_count": 100,
  "page_number": 3,
  "limit": 10,
  "items": [...]
}
```

### カーソルベース方式（NoSQLの場合）

```
GET /users?cursor=eyJpZCI6MTIzfQ&limit=10
```

## 認証・認可

### システム間連携

- **クライアントクレデンシャルフローを推奨**: OAuth 2.0 Client Credentials Flow
- アクセストークンを`Authorization: Bearer {token}`ヘッダで送信

### フロントエンド認証

- **Authorization Code Flow with PKCE**: OAuth 2.0認可コードフロー + PKCE
- アクセストークンは`Authorization`ヘッダで送信

## 楽観ロック

バージョン番号方式を採用:
- 一覧取得のボディからバージョンを取得
- リクエストボディにバージョンを設定
- 更新が競合した場合、409 Conflictを返す

DELETEメソッドの場合は、クエリパラメータに`lock_no`を指定:
```
DELETE /items/12345?lock_no=6192
```

## ファイル連携

### ファイルアップロード

- **小さなデータ（数KB程度）**: Base64でエンコードしてJSONに含める
- **それ以外**: 署名付きURLを利用（S3など）

### ファイルダウンロード

- **サムネイルなど**: Base64でエンコードし、データURIスキームで返す
- **それ以外**: 署名付きURLを利用

## バッチAPI

複数のリソースを同時に処理する場合:

```
POST /users/batch
```

リクエストボディ:
```json
{
  "items": [
    { "id": 1, "name": "User 1" },
    { "id": 2, "name": "User 2" }
  ]
}
```

エラーが発生したリソースの一覧とエラーを返す（成功したリソースは返さない）

## 設計時のチェックリスト

### 目的/スコープ
- [ ] 利用者とユースケースが明確か
- [ ] 対象範囲（読み/書き/検索/管理）が明示されているか
- [ ] 非機能要件（SLA/監査/レート/キャッシュ）が共有されているか

### リソース設計
- [ ] リソース名は複数形でkebab-caseか
- [ ] ネスト/フラットの判断理由があるか
- [ ] カスタムメソッドは最小限で、命名が一貫しているか
- [ ] サブリソース/集約の責務が分離されているか

### HTTPメソッド/冪等性
- [ ] CRUDに対して適切なメソッドが選ばれているか
- [ ] 冪等性が必要な操作でPOSTを使っていないか
- [ ] 変更系はPUT/PATCHのルールが決まっているか

### バージョニング/互換性
- [ ] パス方式でバージョニングしているか
- [ ] 互換性を壊す変更の条件が明記されているか
- [ ] 同一バージョン内の追加/変更ルールがあるか

### リクエスト/レスポンス
- [ ] 入力の必須/任意が明示されているか
- [ ] フィールド命名規則が統一されているか
- [ ] ページングの形と既定値が定義されているか
- [ ] レスポンスの最小構成と例があるか

### エラーハンドリング
- [ ] 代表的な失敗ケースが列挙されているか
- [ ] HTTPステータスの使い分けが一貫しているか
- [ ] エラーボディのスキーマと例があるか

### セキュリティ/運用
- [ ] 認証・認可の方式が明記されているか
- [ ] 監査/ログ方針があるか
- [ ] レートリミット/スロットリング方針があるか
- [ ] キャッシュ方針があるか（no-store含む）

### パフォーマンス/拡張性
- [ ] N+1や過大レスポンスへの配慮があるか
- [ ] フィールド選択やページサイズの上限があるか
- [ ] 将来拡張の余地（任意項目の追加）があるか

### 非同期処理
- [ ] 同期/非同期の判断基準が明記されているか
- [ ] 202 Accepted を使う場合のフォローアップ手段があるか（例: ステータス取得API）

#### 非同期APIの最小例

**処理開始**
```
POST /jobs
```

レスポンス:
```json
{
  "job_id": "job_123",
  "status": "queued",
  "status_url": "/jobs/job_123"
}
```

**状態取得**
```
GET /jobs/{job_id}
```

レスポンス（完了時）:
```json
{
  "job_id": "job_123",
  "status": "succeeded",
  "result": {
    "items": []
  }
}
```

レスポンス（失敗時）:
```json
{
  "job_id": "job_123",
  "status": "failed",
  "error": {
    "code": "PROCESSING_ERROR",
    "message": "failed to process job"
  }
}
```

## 参考資料

- [GitHub REST API Documentation](https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api)
- [フューチャー株式会社 Web API設計ガイドライン](https://future-architect.github.io/arch-guidelines/documents/forWebAPI/web_api_guidelines.html)
- [Google Cloud API Design Guide](https://docs.cloud.google.com/apis/design)
