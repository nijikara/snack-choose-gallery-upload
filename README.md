
# お菓子二択ギャラリー Web アプリ マニュアル

このドキュメントは、以下の機能を持つ Web アプリのマニュアルです。

- Supabase + GitHub Pages で動く **静的フロントエンドのみ** のアプリ
- 「お菓子二択」スタイルのギャラリー（複数ペア）を作成・共有
- 画像つき or 画像なしテキストのみ、どちらでも OK
- 投票機能（左／右の票数と **％表示**）
- 投票結果リセット機能
- 投票せずに結果だけ見る機能
- 投票者の「お名前」を任意入力（軽い連投チェックに利用）
- Supabase Storage への画像アップロード（リサイズあり）
- ギャラリーごとの完全削除（DB + Storage 画像）
- この端末ごとのギャラリー履歴（localStorage）

`index.html` 1 枚＋ Supabase ＋ GitHub Pages だけで完結する構成を想定しています。

---

## 1. 全体構成と用語

### 1.1 全体構成

- **フロントエンド**：`index.html`
  - JavaScript 内で Supabase JS SDK を利用
  - GitHub Pages などの静的ホスティングで配信
- **バックエンド**：Supabase
  - Postgres テーブル：`galleries`, `votes`
  - Storage バケット：`snack-images`（例）
  - Row Level Security（RLS）で anon でも読み書きできるようポリシー設定

### 1.2 主な用語

- **ギャラリー（gallery）**  
  一つのアンケートセット。タイトルと複数の「ペア（対戦カード）」を持つ。  
  例：「部署飲み会お菓子アンケート」

- **ペア（pair）**  
  ギャラリー内の 1 対決。  
  例：「アルフォート ミルクチョコ vs リッチミルク」

- **左／右（left/right）**  
  各ペアの片側。ラベルと画像（任意）を持つ。  
  画像がない場合はテキストのみの枠で表示される。

- **投票（vote）**  
  1 人のユーザーが「このギャラリーに対して送信した選択セット」。  
  ユーザーは任意で「お名前」を入力できる。

---

## 2. 必要なもの

- GitHub アカウント（GitHub Pages でホストする場合）
- Supabase アカウント
- テキストエディタ（VS Code など）
- ブラウザ（Chrome, Edge 等）

---

## 3. Supabase の準備

### 3.1 プロジェクト作成

1. [Supabase](https://supabase.com/) にログイン
2. 新規プロジェクトを作成
3. プロジェクト作成後、**Project Settings → API** を開き、以下を控える
   - **Project URL**（例：`https://xxxx.supabase.co`）
   - **anon public key**（クライアントから使ってよい公開キー）

> ⚠️ `service_role` キーはブラウザから絶対に使わないこと。

---

### 3.2 Storage バケット作成

1. 左メニュー **Storage** を開く
2. **Create bucket** をクリック
3. 例として以下のように設定
   - Bucket name: `snack-images`
   - Public bucket: ON（公開バケット）
4. 作成する

以降、このマニュアルでは `snack-images` を例として使用します。

---

### 3.3 テーブル `galleries` 作成

#### 3.3.1 スキーマ

`galleries` テーブルはギャラリー本体を保存します。

- **Schema**: `public`
- **Table name**: `galleries`
- **RLS**: 有効（ON）
- カラム定義（例）:

| カラム名    | 型           | 説明                              |
|------------|--------------|-----------------------------------|
| id         | uuid         | プライマリキー（アプリ側で UUID 生成） |
| title      | text         | ギャラリーのタイトル              |
| config     | jsonb        | ギャラリー全体の設定・ペア情報    |
| created_at | timestamptz  | 作成日時（デフォルト `now()`）    |

SQL で作成する場合の例：

```sql
create table public.galleries (
  id uuid primary key,
  title text,
  config jsonb not null,
  created_at timestamptz default now()
);

alter table public.galleries enable row level security;
```

#### 3.3.2 RLS ポリシー

ブラウザ（anonロール）からの `select / insert / delete` を許可します。

```sql
-- SELECT を許可
create policy "public select galleries"
on public.galleries
for select
to anon
using (true);

-- INSERT を許可
create policy "public insert galleries"
on public.galleries
for insert
to anon
with check (true);

-- DELETE を許可（完全削除機能で使用）
create policy "public delete galleries"
on public.galleries
for delete
to anon
using (true);
```

---

### 3.4 テーブル `votes` 作成

投票結果を保存するテーブルです。

#### 3.4.1 スキーマ

```sql
create table public.votes (
  id uuid primary key default gen_random_uuid(),
  gallery_id uuid not null references public.galleries(id) on delete cascade,
  pair_index integer not null,
  choice text not null check (choice in ('left','right')),
  voter_name text,
  created_at timestamptz default now()
);

alter table public.votes enable row level security;
```

- `gallery_id` : どのギャラリーへの投票か
- `pair_index` : 何番目のペアか（1, 2, 3, ...）
- `choice` : `'left'` または `'right'`
- `voter_name` : 投票者名（任意・null 可）
- `on delete cascade` により、ギャラリー削除時にそのギャラリーの投票も自動削除されます。

#### 3.4.2 RLS ポリシー

```sql
-- 結果の読み取り
create policy "public select votes"
on public.votes
for select
to anon
using (true);

-- 投票の追加
create policy "public insert votes"
on public.votes
for insert
to anon
with check (true);

-- 投票のリセット（削除）
create policy "public delete votes"
on public.votes
for delete
to anon
using (true);
```

---

### 3.5 Storage（`storage.objects`）用 RLS

画像のアップロード・閲覧・削除に必要です。

```sql
-- 閲覧
create policy "public read snack-images"
on storage.objects
for select
to anon
using (bucket_id = 'snack-images');

-- アップロード
create policy "public upload snack-images"
on storage.objects
for insert
to anon
with check (bucket_id = 'snack-images');

-- 削除（完全削除機能で使用）
create policy "public delete snack-images"
on storage.objects
for delete
to anon
using (bucket_id = 'snack-images');
```

> バケット名を `snack-images` 以外にした場合は、`bucket_id` を合わせて変更してください。

---

## 4. `index.html` の配置と設定

### 4.1 ファイル配置

1. 任意のフォルダを作成（例：`snack-gallery`）
2. その中に `index.html` を配置
3. ChatGPT で生成した最新版の HTML コードを貼り付け

### 4.2 APP_CONFIG の設定

`index.html` 内の `APP_CONFIG` を Supabase プロジェクトに合わせて変更します。

```js
const APP_CONFIG = {
  supabase: {
    url: "https://YOUR-PROJECT.supabase.co",
    anonKey: "YOUR-ANON-KEY",
    bucketName: "snack-images",
    votesTable: "votes",
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

- `url` : Supabase Project URL
- `anonKey` : anon public key
- `bucketName` : 作成したバケット名
- `votesTable` : 投票テーブル名（デフォルト `votes`）
- 画像の最大サイズ（px）と JPEG 品質を `image` セクションで調整可能
- `history.maxMyGalleries` はこのブラウザで保持するギャラリー履歴の最大件数

---

## 5. GitHub Pages へのデプロイ（任意）

1. GitHub で新しいリポジトリを作成（例：`snack-gallery`）
2. `index.html` をルートに push
3. リポジトリの **Settings → Pages** で
   - Source: `main`（または `master`）
   - Folder: `/ (root)`
   を選択して保存
4. しばらくすると、
   - `https://<ユーザー名>.github.io/<リポジトリ名>/`
   で `index.html` が公開される

---

## 6. 画面構成

### 6.1 作成済みギャラリー一覧（この端末）

- このブラウザで作成・保存したギャラリーの履歴を表示
- localStorage (`mySnackGalleries`) に保存されている内容をもとに描画
- 各ギャラリー行に以下のボタンがある：
  - **開く（投票画面）** : `...?gallery=ID` を別タブで開く
  - **編集する（複製）** : `...?edit=ID` としてビルダーに読み込む
  - **URLコピー** : 共有 URL をクリップボードにコピー
  - **完全削除（DB+画像）** : Supabase 上の `galleries` レコード・関連画像を削除し、localStorage からも履歴を削除

> 「完全削除」は元データ・画像をすべて削除します。取り消し不可。

### 6.2 ビルダー画面（ギャラリー作成・編集）

- `?edit` パラメータの有無でモードが変わる
  - パラメータなし：**新規作成モード**
  - `?edit=ギャラリーID`：**編集モード（複製として新規保存）**
- 入力項目：
  - ギャラリータイトル
  - 任意個数の「ペア」

各ペアには以下を設定：

- ペアタイトル（例：アルフォート）
- 左ラベル（例：ミルクチョコ）
- 左画像（任意）
- 右ラベル（例：リッチミルク）
- 右画像（任意）

その他：

- 「＋ ペアを追加」ボタンで対決カードを増やす
- **「このペアを削除」ボタン**で任意のペアを削除可能
- ペア削除後は自動で「ペア 1, 2, 3,...」と番号が振り直される

**画像について：**  
画像は任意です。

- 両方画像あり → 通常の画像対決
- 一方または両方画像なし → テキストのみボックスで表示

### 6.3 ビューア画面（投票）

`...?gallery=ギャラリーID` で表示される画面です。

- 画面上部：
  - ギャラリータイトル
  - 「お名前（任意）」入力欄
- 中央：
  - 各ペアのカード
- 下部：
  - 「この投票を送信する」
  - 「投票結果をリセット」
  - 「投票せずに結果を見る」

---

## 7. ギャラリー作成の流れ

1. `index.html`（または GitHub Pages の URL）にアクセス
2. 画面上部の「① ギャラリー作成」で
   - タイトルを入力
   - 必要な数だけペアを追加し、タイトル・ラベル・（必要であれば）画像を設定
3. 入力完了後、**「ギャラリー保存＆共有URLを生成」** を押す
4. Supabase に以下が保存される：
   - `galleries.id` : UUID
   - `galleries.title` : ギャラリータイトル
   - `galleries.config` : ペアと画像 URL を含む JSON
5. 画面下部に共有 URL（`...?gallery=ID`）が表示される
6. 同時に localStorage の「作成済みギャラリー一覧」にも履歴が追加される

### 7.1 画像アップロードとリサイズ

- ファイル選択時、JavaScript で画像をリサイズ：
  - 最大サイズ：`maxWidth` × `maxHeight`（デフォルト 200 × 200）
  - フォーマット：JPEG
  - 品質：`quality`（デフォルト 0.8）
- リサイズ済み画像を Supabase Storage の指定バケットにアップロード
- アップロードパス例：  
  `galleryId/1-left-タイムスタンプ.jpg`
- `config` には以下を保存：
  - `imageUrl` : 公開 URL
  - `storagePath` : バケット内パス（削除時に使用）

画像なしの場合：

- ファイル未選択で保存すると `imageUrl` は `null`、`storagePath` も `null`
- viewer ではテキストのみの箱として表示される

---

## 8. ギャラリー編集（複製）

### 8.1 編集モードへの入り方

- 「作成済みギャラリー一覧」の行で「編集する（複製）」を押す
- URL が `...?edit=ギャラリーID` になり、ビルダー画面が
  - ギャラリータイトル
  - 既存のペア
  - 画像プレビュー
  付きで読み込まれる

### 8.2 編集〜新規保存

- テキスト修正・画像差し替え・ペア削除・ペア追加など任意に行う
- 「ギャラリー保存＆共有URLを生成」を押すと：
  - **新しい UUID** で `galleries` に `insert`
  - 共有 URL も新しく発行
  - 「作成済みギャラリー一覧」にも新しいギャラリーとして追加

元のギャラリーはそのまま残ります。

---

## 9. 投票機能の詳細

### 9.1 投票画面の操作

1. 共有 URL（`...?gallery=ID`）を開く
2. 必要であれば、上部の「お名前（任意）」に投票者名を入力
   - 入力した名前は localStorage に保存され、同じブラウザで再利用される
3. 各ペアについて、左または右のいずれかをクリック（タップ）
   - 選択した側に枠が付き、ハイライト表示
   - 選択は何度でも切り替え可能（最後の状態が採用される）
4. 投票を確定するときは **「この投票を送信する」** を押す

### 9.2 投票送信時の挙動

- 送信ボタン押下時：
  - `currentSelections` に入っているペアだけが投票として送信される
  - 1 ペアも選んでいない場合はエラー表示
- 軽い連投チェック：
  - localStorage に `snackVoted_<galleryId>` というフラグが残る
  - 2 回目以降の投票時には
    - 「このギャラリーにはすでに投票済みの可能性があります。もう一度投票しますか？」
    - という確認ダイアログが表示される（キャンセル可能）
- votes テーブルへの保存内容：
  - `gallery_id` : 対象ギャラリー ID
  - `pair_index` : ペア番号（1, 2, ...）
  - `choice` : `'left'` or `'right'`
  - `voter_name` : 名前欄に入力した文字列（未入力なら null）

### 9.3 投票結果の表示（％計算）

投票送信後、または「投票せずに結果を見る」押下後に、  
`votes` テーブルから対象ギャラリーの全投票を集計します。

- 各ペアについて：
  - `leftVotes` : `choice='left'` の件数
  - `rightVotes` : `choice='right'` の件数
  - `total = leftVotes + rightVotes`
- パーセンテージ計算：
  - `leftPct = round(leftVotes / total * 100)`
  - `rightPct = round(rightVotes / total * 100)`

表示例：

```text
左: 3票 (60%) / 右: 2票 (40%)（合計 5票）
```

投票が 0 件の場合：

```text
まだ投票はありません。
```

### 9.4 結果表示タイミング

- ページ読み込み直後：
  - 各ペアの下には
    - 「※ 投票を送信するか『投票せずに結果を見る』を押すと結果が表示されます。」
    - という説明文のみ表示
  - 票数や％は表示されない
- 「この投票を送信する」押下後：
  - insert 成功後に集計を読み込み、票数・％を表示
- 「投票せずに結果を見る」押下後：
  - insert はせず、既存の votes のみで集計表示

---

## 10. 投票結果のリセット

### 10.1 リセットボタン

ビューア画面の下部にある **「投票結果をリセット」** ボタンで、  
そのギャラリーに紐づく `votes` を全削除できます。

### 10.2 挙動

1. 確認ダイアログ表示
2. `votes` テーブルに対して：

```sql
delete from public.votes
where gallery_id = :current_gallery_id;
```

3. リセット後：
   - 各ペアのサマリ表示は
     - 「投票はリセットされました。再度投票するか『投票せずに結果を見る』で表示できます。」
   - localStorage の `snackVoted_<galleryId>` も削除（連投フラグ解除）

---

## 11. 完全削除（DB + 画像）

### 11.1 ボタンの配置

- 「作成済みギャラリー一覧（この端末）」の各ギャラリー行に  
  **「完全削除（DB+画像）」** ボタンがある

### 11.2 挙動の流れ

1. `galleries` から、対象 ID の `config` を取得
2. `config.pairs[*].left.storagePath` / `right.storagePath` を収集
   - `storagePath` がない古いデータの場合は、`imageUrl` からパスを推測
3. Supabase Storage の削除 API：

```js
sb.storage.from(bucketName).remove(paths);
```

4. `galleries` から対象レコードを削除：

```js
delete from public.galleries where id = :id;
```

5. `votes.gallery_id` は `on delete cascade` により自動的に削除
6. localStorage の履歴からも該当ギャラリーを削除

> この操作は元に戻せません。注意して実行してください。

---

## 12. localStorage の使い道

ブラウザごとに、以下の情報を localStorage に保存しています。

- `mySnackGalleries`  
  このブラウザで作成・保存したギャラリー一覧  
  （ID, タイトル, 作成日時）

- `snackVoterName`  
  viewer 上部の「お名前（任意）」入力欄の内容  
  → 次回同ギャラリー以外でも自動入力される

- `snackVoted_<galleryId>`  
  そのギャラリーに一度投票したことがあるかどうかのフラグ  
  → 2 回目以降の投票時に警告ダイアログ表示に利用

これらはユーザー自身のブラウザ内だけで完結し、サーバーには保存されません。

---

## 13. トラブルシューティング

### 13.1 `new row violates row-level security policy`

RLS によって操作がブロックされている場合の典型的なエラーです。

- 画像アップロードで出る場合：
  - `storage.objects` の `insert / delete / select` ポリシーを確認
  - バケット名 (`bucket_id`) が実際のバケット名と一致しているか

- ギャラリー保存で出る場合：
  - `galleries` テーブルの `insert / select / delete` ポリシーを確認

- 投票関連で出る場合：
  - `votes` テーブルの `insert / select / delete` ポリシーを確認

### 13.2 画像が削除されない

完全削除実行後も画像が Storage に残っている場合：

- `storagePath` の保存・復元処理に問題がないか確認
- 古いギャラリーで `storagePath` がない場合、`imageUrl` からのパス推測がうまくいっているか
- Storage の delete ポリシーが正しく設定されているか

### 13.3 GitHub Pages で Supabase に接続できない

- `APP_CONFIG.supabase.url`／`anonKey` が正しいか再確認
- ブラウザの DevTools の Network タブで CORS エラーが出ていないか確認
- ローカルで `file:///` 経由で開いた場合、挙動が不安定になることがあるので、基本は HTTP 経由（簡易ローカルサーバーか GitHub Pages）で確認する

---

## 14. 拡張アイデア（メモ）

今後追加を検討できる機能例：

- ペア順のランダム出題
- 投票完了後の「あなたの傾向」診断表示（甘党／しょっぱい派 など）
- 接戦ランキング（差が少ないペア順に並べる）
- 投票締切日時付きギャラリー
- 結果の CSV ダウンロード

実装時には、本マニュアルにセクションを追加していく運用を想定しています。

---

以上が、現状の実装を前提とした `.md` 形式のマニュアルです。  
`README.md` としてリポジトリのルートに置くと、あとから見返しやすくなります。
