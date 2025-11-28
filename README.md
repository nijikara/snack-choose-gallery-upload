
# お菓子二択ギャラリー（Supabase版）マニュアル

このドキュメントは、  
「お菓子の二択バトル画像ギャラリー」を作って共有できる Web アプリ  
（`index.html` 1枚＋Supabase＋GitHub Pages）向けのマニュアルです。

- 画像アップロード → Supabase Storage に保存
- ギャラリー設定 → Supabase Postgres に保存
- 共有URL → `...?gallery=ID`
- 編集 → 既存ギャラリーを複製して新URL作成
- 履歴 → ブラウザごとに localStorage で管理
- 削除 → Supabaseごと完全削除（オプション）

を前提としています。

---

## 1. 前提・必要なもの

- GitHub アカウント（GitHub Pages を使う）
- Supabase アカウント
- 任意のテキストエディタ（VSCode 等推奨）
- ブラウザ（Chrome / Edge など）

---

## 2. Supabase 側の準備

### 2.1 プロジェクト作成

1. [Supabase](https://supabase.com/) にログイン
2. 新規プロジェクトを作成
3. プロジェクト作成後、**Project Settings → API** を開き、以下をメモ：

   - **Project URL**（例：`https://xxxx.supabase.co`）
   - **anon public key**（`anon` と書かれている公開キー）

> ⚠️ `service_role` キーはフロントでは絶対に使わないこと

---

### 2.2 Storage バケット作成

1. 左メニュー **Storage** を開く
2. **Create bucket** をクリック
3. 以下のように設定：

   - Bucket name: `snack-images`（任意だが、後述設定と揃える）
   - Public bucket: **ON**（公開バケットにする）

4. 作成する

---

### 2.3 `galleries` テーブル作成

1. 左メニュー **Table editor** → **New table**
2. 以下で作成（SQL で作る場合は次の節参照）：

   - Schema: `public`
   - Table name: `galleries`
   - RLS: 有効（Row Level Security ON）
   - カラム：
     - `id` : `uuid`, Primary key
     - `title` : `text`, Nullable 可
     - `config` : `jsonb`, Not null
     - `created_at` : `timestamptz`, Default: `now()`

### 2.4 `galleries` 用 RLS ポリシー

SQL Editor で以下を実行すると、  
`galleries` への `SELECT / INSERT / DELETE` を anon ロールに開放できます。

```sql
-- SELECT を anon に許可
create policy "public select galleries"
on public.galleries
for select
to anon
using (true);

-- INSERT を anon に許可
create policy "public insert galleries"
on public.galleries
for insert
to anon
with check (true);

-- DELETE を anon に許可（完全削除機能を使う場合）
create policy "public delete galleries"
on public.galleries
for delete
to anon
using (true);
````

※ 開発中は `galleries` の RLS を OFF にしても動きますが、
　本番運用を意識するなら ON ＋ポリシーが推奨です。

---

### 2.5 Storage（`storage.objects`）用 RLS ポリシー

画像アップロード・閲覧・削除のために、`storage.objects` にもポリシーが必要です。

SQL Editor で以下を実行：

```sql
-- snack-images バケット内の画像の「閲覧」を anon に許可
create policy "public read snack-images"
on storage.objects
for select
to anon
using (bucket_id = 'snack-images');

-- snack-images バケット内の画像の「アップロード」を anon に許可
create policy "public upload snack-images"
on storage.objects
for insert
to anon
with check (bucket_id = 'snack-images');

-- snack-images バケット内の画像の「削除」を anon に許可（完全削除機能を使う場合）
create policy "public delete snack-images"
on storage.objects
for delete
to anon
using (bucket_id = 'snack-images');
```

> ⚠️ `bucket_id` はバケット名に合わせて変更してください
> 本マニュアルでは `snack-images` を前提にしています。

---

## 3. `index.html` の設置

1. 手元で任意のフォルダを作成（例：`snack-gallery`）
2. その中に `index.html` を作り、チャットで渡された最新版コードを貼り付け
3. コード内の `APP_CONFIG` を自分の環境に合わせて修正：

```js
const APP_CONFIG = {
  supabase: {
    // ★ここを自分の Supabase プロジェクトに合わせる
    url: "https://YOUR-PROJECT.supabase.co",
    anonKey: "YOUR-ANON-KEY",
    bucketName: "snack-images",
  },
  image: {
    maxWidth: 200,
    maxHeight: 200,
    mime: "image/jpeg",
    quality: 0.8,
  },
  history: {
    maxMyGalleries: 50,
  },
};
```

* `url`：Supabase の Project URL
* `anonKey`：`anon public` キー
* `bucketName`：作成した Storage バケット名（例：`snack-images`）
* `image.maxWidth / maxHeight`：アップロード時に縮小する最大サイズ（px）
* `history.maxMyGalleries`：このブラウザに保持するギャラリー履歴の最大件数

---

## 4. GitHub Pages へのデプロイ

### 4.1 リポジトリ作成

1. GitHub で新しいリポジトリを作成（例：`snack-gallery`）
2. `index.html` をリポジトリのルートに配置して push

### 4.2 GitHub Pages 設定

1. GitHub のリポジトリページを開く

2. 上部の **Settings** → 左メニュー **Pages**

3. **Source** を設定：

   * Branch: `main`（または `master`）
   * Folder: `/ (root)`

4. 保存すると、少し時間をおいて

   * `https://ユーザー名.github.io/リポジトリ名/`

   で `index.html` が公開されます。

---

## 5. アプリの使い方

### 5.1 画面構成

* **作成済みギャラリー一覧（この端末）**

  * このブラウザで「保存」したギャラリーの履歴（localStorage）
  * 開く／編集（複製）／URLコピー／完全削除
* **ギャラリー作成・編集画面**

  * 新規作成 or 既存ギャラリーを読み込んで編集（複製）
* **ギャラリー閲覧画面**

  * `...?gallery=ID` で開く閲覧・投票用画面

---

### 5.2 ギャラリーを新規作成する

1. `index.html`（GitHub Pages の URL）を通常アクセスする
   → クエリがない状態（例：`...?gallery=xxx` が付いていない）
2. 「① ギャラリー作成（新規）」カードが表示される
3. 入力項目：

   * ギャラリータイトル（例：「お菓子二択バトル」）
   * 「＋ ペアを追加」ボタンを押す

     * ペアタイトル（例：アルフォート）
     * 左ラベル（例：ミルクチョコ）
     * 左画像ファイル（ローカル画像）
     * 右ラベル（例：リッチミルク）
     * 右画像ファイル
   * ペアは複数追加可能
4. 入力が終わったら
   → **「ギャラリー保存＆共有URLを生成」** を押す
5. 保存成功すると、下部に「このURLを友だちに送ってください」として
   `...?gallery=UUID` 形式の共有URLが表示される
6. 同時に「作成済みギャラリー一覧」にも履歴として追加される

> 補足：画像はアップロード時に最大 200x200 の JPEG にリサイズされてから
> Supabase Storage に保存されます。

---

### 5.3 共有URLを送って使ってもらう

* 5.2 で表示された URL をコピーして、友人に送る
* 相手はそのURLを開くだけで、
  → 画像＋ラベル付きの「どっち派？」画面が表示される
* 画像をタップ／クリックすると、選んだほうが枠で選択状態になる
  （このサンプルでは「選択状態をその場で楽しむ」モードで、投票結果はサーバー保存していません）

---

### 5.4 作成済みギャラリー一覧（この端末）

トップの「作成済みギャラリー一覧」には、
**このブラウザで保存したギャラリー**が表示されます（localStorage）。

1件ごとに以下の情報・ボタンがあります：

* タイトル
* ID / 作成日時
* **開く（投票画面）**

  * 別タブで `...?gallery=ID` を開く
* **編集する（複製）**

  * `...?edit=ID` として開く（ビルダーに読み込み）
  * 編集後に保存すると「新しいID」としてギャラリーが作成される
* **URLコピー**

  * `...?gallery=ID` のURLをクリップボードにコピー
* **完全削除（DB+画像）**

  * Supabase 上の `galleries` レコード＋Storage の画像ファイルも削除
  * 共有URLからもアクセス不可になる
  * 最後に localStorage の履歴も削除

---

### 5.5 ギャラリーを編集（複製）する

1. 「作成済みギャラリー一覧」の対象の行で **「編集する（複製）」** を押す
   → URL が `...?edit=ID` の形で読み込まれる
2. ギャラリー作成画面に、既存の：

   * ギャラリータイトル
   * 各ペアのタイトル
   * ラベル
   * 画像プレビュー
     が自動でセットされる
3. テキストを修正したり、一部の画像を差し替えたりして編集
4. 「ギャラリー保存＆共有URLを生成」を押すと：

   * 新しい UUID が採番され
   * 新規ギャラリーとして Supabase に INSERT
   * 新しい共有URLが発行される
   * 「作成済みギャラリー一覧」にも新しいギャラリーとして追加される

> 元のギャラリーは残り、新旧2つのURLが並存します。

---

### 5.6 完全削除（DB＋画像）について

「作成済みギャラリー一覧」の「完全削除（DB+画像）」ボタンは、

1. `galleries` から対象IDの `config` を取得
2. `config.pairs[].left.imageUrl / right.imageUrl` から
   Supabase Storage のパスを抽出
3. Storage から該当パスのファイルを削除
4. `galleries` から該当行を DELETE
5. localStorage の履歴も削除

という流れで、「そのギャラリーに関する全てのサーバー側データ」を削除します。

> この操作は取り消せません。
> 本当に不要なギャラリーだけに使ってください。

---

## 6. 設定を変えたいとき

### 6.1 画像サイズ・品質を変える

`APP_CONFIG.image` を編集：

```js
image: {
  maxWidth: 200,
  maxHeight: 200,
  mime: "image/jpeg",
  quality: 0.8,
},
```

* 例：画像をもう少し大きくしたい → `maxWidth / maxHeight` を 400 にする
* 例：もっと軽くしたい → `quality` を 0.7 に下げる

### 6.2 履歴件数を変える

```js
history: {
  maxMyGalleries: 50,
},
```

* 例：履歴を10件だけにしたい → `10` に変更

---

## 7. よくあるトラブルと対処

### 7.1 「new row violates row-level security policy」が出る

RLS（Row Level Security）に引っかかっているときのエラーです。

* 画像アップロード時に出る場合

  * `storage.objects` の `INSERT / DELETE / SELECT` ポリシーが不足している
  * このマニュアル「2.5 Storage用 RLS ポリシー」を再確認
* ギャラリー保存時（DB挿入）に出る場合

  * `galleries` の `INSERT / SELECT / DELETE` ポリシーが不足している
  * このマニュアル「2.4 galleries 用 RLS ポリシー」を再確認

開発中にどうしても切り分けたい場合は、

* 一時的に `galleries` の RLS を OFF にして挙動を見る
* Storage 側のポリシーを別途調整する

など、原因のテーブルを切り分けると理解しやすいです。

---

## 8. 拡張のアイデア

このサンプルは「投票結果は保存せず、選択状態をその場で楽しむ」仕様ですが、
さらにやりたくなってきたときに考えられる拡張：

* 投票結果を保存する `votes` テーブルを追加

  * `gallery_id`, `pair_index`, `choice`, `session_id`, `created_at` など
* 集計画面を追加して、「何人がどっちを選んだか」を可視化
* 1ギャラリーあたりの最大ペア数を `APP_CONFIG` で制限
* ギャラリー作成者を識別する簡易な認証（Supabase Auth）　など

---

## 9. ライセンス・注意事項

* このアプリは、あくまで個人的な遊び・企画用途を想定しています。
* 画像は著作権などに配慮し、利用してよいものだけをアップロードしてください。
* Supabase の無料枠を超えないよう、ときどき Storage の容量を確認し、
  不要なギャラリーは「完全削除」で掃除することをおすすめします。
