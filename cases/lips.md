# LIPS Scraping Design（Octoparse）

## 目的
最新投稿一覧から投稿を複数件収集し、投稿詳細→ユーザープロフィールへ辿ってSNSリンクを抽出する。
運用上「欠損は空欄が正」とし、誤取得より欠損を優先する。

---

## 対象ページ
- 一覧: https://lipscosme.com/posts?sort=latest
- 投稿詳細: https://lipscosme.com/posts/{post_id}
- プロフィール: https://lipscosme.com/users/n/@{username}（例）

---

## 全体フロー
1. 一覧ページで投稿カードをLoop（1ページあたり約20件）
2. 各投稿で投稿詳細へ遷移
3. 投稿日時 / 投稿URL / ユーザーURL を取得
4. ユーザーURLへ遷移
5. SNSリンク（Instagram / X / YouTube / TikTok / Web）を取得
6. 一覧を次ページへ（最大10ページ）

---

## Octoparse設計（推奨）
### Loop（投稿カード）
- Loop Item: 投稿カード要素（一覧上の1投稿単位）
- 取得順:
  - 投稿詳細へ遷移（クリック or URL）
  - 詳細でデータ取得
  - プロフィールへ遷移
  - SNS取得
  - 戻る/次の投稿

### Pagination（10ページ取得）
- Next ボタンでページ遷移できる場合：
  - Paginationを有効化し最大10ページ
- Nextが不安定な場合：
  - URLパターンがあるならURLリスト方式（推奨）
  - どちらも不可なら「クリック＋待機＋検証」で運用

---

## 取得項目（XPath / CSS）
※ここが成果物の核。誤取得を避けるため「範囲をプロフィール領域に限定」する。

### 投稿詳細ページ
| Field | 意味 | Selector（確定版を記載） | 備考 |
|---|---|---|---|
| post_url | 投稿URL | `{{AUTO}}` | URL取得でOK |
| published_at | 投稿日時 | `（あなたの確定XPath）` | 動的読み込み対策でDelay必須 |
| user_url | ユーザーURL | `（あなたの確定XPath）` | プロフィールへ遷移用 |

### プロフィールページ（SNS）
| Field | Selector（確定版） | ルール |
|---|---|---|
| instagram_url | `(//*[...profile... ]//a[contains(@href,"instagram.com")])[1]/@href` | 公式（lipsjp等）を拾う場合は範囲をユーザー領域に限定 |
| x_url | `（同様にx.com / twitter.com）` | ない場合は空欄 |
| youtube_url | `（同様にyoutube.com）` | ボタンだけでURLが無いケースに注意 |
| tiktok_url | `（同様にtiktok.com）` | ない場合は空欄 |
| website_url | `（外部ドメインでSNS以外）` | 「Webサイト」ボタンだけでDOMにURLが無い場合は空欄が正 |

---

## 待機（Delay）方針
- 投稿詳細：
  - ページ表示後 `X ms` 待機（例：1500〜3000ms）
- プロフィール：
  - SNSリンクが描画されるまで `Y ms` 待機
- 根拠：
  - 同一URLでも読み込みタイミングにより要素認識が落ちることがあるため

---

## 重複・欠損の扱い
- 欠損（SNS未登録）は **空欄が正**
- 重複排除キー：
  - `post_url` をユニークキーとして運用（推奨）
- 誤取得回避：
  - 「ページ外（ヘッダー/フッター/公式アカウント）」を拾わないようXPathの範囲制限を徹底

---

## 既知の問題と対策
- 一覧→詳細へ遷移後に要素認識が落ちる
  - 対策：クリック/URL遷移の切替、Delay増加、要素範囲の再選択
- プロフィールで「Webサイト」ボタンがあるのにURLが取れない
  - 対策：HTMLにURLが無いなら空欄が正（無理に埋めない）

---

## 出力例（カラム）
post_url, published_at, user_url, instagram_url, x_url, youtube_url, tiktok_url, website_url
