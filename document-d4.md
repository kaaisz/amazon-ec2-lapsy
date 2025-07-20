<link href="/assets/css/style.css" rel="stylesheet" />

[←TOPへ戻る](document.html)

# 自律的情報技術学習演習：作業の記録 Day-4(後半)

DS233292 鈴木可愛

## これは何

ムサビ通信「自律的情報技術学習演習」の3日目以降〜後半日程の授業の記録です
前日までのセットアップの続きのうちCRUD処理のDB連携を実施。徐々に作成済みプロトタイプとの統合に入っていきます
※これ以降中間課題までかなり時間が限られていたので一部ドキュメント内容を割愛したり後ほど更新予定

# 今日やること：既存コードとの連携を開始

ここから先は投稿機能などがメインになってくるが、どのような投稿をする機能なのか、どこと連携するのかなどをAIに理解させ、より実装を早めるために既存プロトタイプのコードを貼っていくことに

# 1. 一時的に`/prototype` ディレクトリを作成して、Figma Makeで生成したコードを格納

[@GitHub: Temporarily added codes for prototype in /prototype directory](https://github.com/kaaisz/lapsy-front/commit/9e694c7194770ace580398957f2781bd6cbfd8b2)

追記：このままではVercelビルド時にエラーで通らないので、
`tsconfig.json`の`"exclude"`に`prototype`を追加

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules", "prototype"]
}
```


# 2. Supabaseと1-1を連携させる

## 2-1. Supabaseにpostsテーブルを作成する

SupabaseのSQLエディタで下記SQL文を実行

```SQL
create table posts (
  id uuid primary key default gen_random_uuid(),
  content text not null,
  postDate timestamptz not null,
  createdAt timestamptz not null default now(),
  updatedAt timestamptz not null default now(),
  isDraft boolean not null default false,
  user_id uuid references auth.users(id) -- 投稿者のID（必要に応じて）
);
```

- idは自動生成のUUID
- user_idは認証ユーザーと紐付ける場合のみ（不要なら削除OK）

<aside>

### 💡 RLS（Row Level Security）はどうする？
本番運用時はRLS（Row Level Security）を有効化し、認証ユーザーのみ自分の投稿を操作できるようにポリシーを追加する。
開発中は一時的にRLSを無効化してもOK。

</aside>

## 2-2. 投稿一覧取得（Read）をSupabase連携に置き換える

### 1. Supabaseクライアントのimport

lib/supabaseClient.tsを使用

[@GitHub: Import Supabase client](https://github.com/kaaisz/lapsy-front/commit/2a95cfa9e0b13d5daa001e0a80f07bb3901e9039)

### 2. 投稿一覧を取得する関数を作成

- 取得場所の例：`AppProvider` や `Timeline` など、投稿一覧を管理している箇所
- 方法：`useEffect` 内でSupabaseからデータを取得。

[@GitHub: Create function to fetch all the posts](https://github.com/kaaisz/lapsy-front/commit/84ba3578de17dc5a344574a8820644e0f89a34d4)

### 3. 既存のuseState<Post[]>の初期値やダミーデータを削除

ここまでのプロセスですでにダミーデータや初期値は削除されているので、ここまでのコミットで発生していたエラー(linterにより検出)を潰す

[@GitHub: Get rid of existing initial or dummy data of useState<Post[]>](https://github.com/kaaisz/lapsy-front/commit/758111e5603c8a6e23e4e7fbd93c7d19609ba103)

[@GitHub: Refactor MyPage component to fetch and display posts](https://github.com/kaaisz/lapsy-front/commit/55a173f6511f318519a951b4621332117616362f)

## 2-3. 新規投稿（Create）をSupabase連携に置き換える

### 1. 投稿作成用のUI（フォーム）を用意

<aside>
☝️ ここから徐々に既存プロトタイプからの移植を開始
</aside>

<br>

1. `/prototype` にある `PostComposer` を `/app/components/PostComposer.tsx` に移動(コピー)
2. `PostComposer.tsx` にいる以下は使用しないので削除
    ```tsx
    // import { useNavigate } from "react-router-dom";
    ```
3. 関連コンポーネントも移動するといくつかインストールを要するエラーが発生するので、以下のnpmパッケージをインストール( `npm install` )

    - `lucide-react`
    - `class-variance-authority`
    - `tailwind-merge`
    - `@radix-ui/react-slot`
    - `@radix-ui/react-label`

4. それでもいくつかのエラーが発生・・・

    ![Vercelのエラー：React.useId](/assets/img/image-d4-1.png)

5. 不要なlineを削除し、`React.useId()`を条件分岐の外側のトップレベルで呼び出すようにする

    [@GitHub: Refactor input and switch components to enable React.useId() in top-level](https://github.com/kaaisz/lapsy-front/commit/1ad0468c08cd982e1d71b8c6afce8fe52706d255#diff-957cef4e6ea7c8c8dc7abc51a089cc0e50988e580bf4e15e35dc5b304d1a99b0)

6. 5.で間違えて `/prototype` のコードまで修正しており、ミスを誘発しそうだったので`.gitignore`内で `/prototype` を除外

    [@GitHub: Revert prototype changes before adding .gitignore](https://github.com/kaaisz/lapsy-front/commit/63f63cf9050543d20595dacc369d90d9d8522ff2)
    
    [@GitHub: Remove prototype directory and related files](https://github.com/kaaisz/lapsy-front/commit/dbfe575bff16a9266360f376f68f91f9fe491b07)

7. 合わせて、エラーを吐き出していたコードを一時的にdisableにする

    [@GitHub: Temp disable handleViewDrafts](https://github.com/kaaisz/lapsy-front/commit/9d821a8fc9a475e0d991bdfd1708fa61b9cfa729)

### 2. 投稿作成時にSupabaseへinsert

#### 1. `/app/components/PostComposer.tsx`を `/app/mypage/page.tsx` の中へimport (ログイン成功後の `/mypage` 内で投稿を行いたい)

mypage/page.tsxへの`PostComposer`の設置に成功したが、新規投稿ではなく「投稿を編集」になっている・・・

![デプロイ後のマイページ](/assets/img/image-d4-2.png)

しかもStyleが当たっていないのでとても見づらい

#### 2. globals.cssをfix (直らないので後ほど再訪)

1. `/prototype/styles/globals.css` の内容を `/styles/globals.css` に移植

2. `@custom-variant` や `@theme` がエラーを吐き出している。Figma Makeで作成したStyleは [tailwind](https://tailwindcss.com/) と [shadcn](https://ui.shadcn.com/) でできているので、最新のアップデートに基づいてfixする。<br>※数ヶ月間が開くだけでも使えなくなる変数もあるらしい😓
    <br>

    [@GitHub: ]()
    
    <br>
    以下、参考リンク

    - [Tailwind CSSとは？概要や特徴について簡単に解説！- EVOWORX Blog](https://evoworx.dev/blog/what-is-tailwindcss/)

    - [「shadcn/ui」って何が凄いの？実装知らないWebデザイナーが調べてみた｜akane - note](https://note.com/akane_desu/n/n1276d86d388e)

    - [「スピード」と「こだわり」を両立したい！shadcn/uiとTailwind CSSを活用したゼロからのコンポーネントライブラリ構築 - 10X Product Blog](https://product.10x.co.jp/entry/2024/12/17/161718)

    <br>

3. しかしこれでもまだエラーが出ている。<br>
`tailwindcss`がインストールされていると思いきやされていない可能性があるらしいので、再度トライ

    ```
    npm install tailwindcss postcss autoprefixer
    ```
4. さらに、設定ファイル `tailwind.config.js` がなかったので、追加するために以下のコマンドを実行

    ```
    npx tailwindcss init -p
    ```
    ・・・実行できずにエラーが返ってきてしまった。
5. `node_modules`が壊れているようなので、`package-lock.json`も含めまるごと削除して再度`npm install`でトライ

    ```shellscript
    rm -rf node_modules package-lock.json
    npm install
    ```

6. ・・・ここからハマり始める😇
    もともと入っていた `"@tailwindcss/postcss"`を削除
    ```
    npm uninstall @tailwindcss/postcss
    ```

7. Tailwind CSS v4.x 以降の仕様で、v4.x以降は **`tailwindcss-cli` というパッケージが別で必要** らしいので、追加インストール
    ```
    npm install -D tailwindcss-cli
    ```

    その後、下記コマンドを実行
    ```
    npx tailwindcss-cli init -p
    ```

8. これで `postcss.config.js` と `tailwind.config.js` が追加 + 初期化された！

    ![追加に成功した画面](/assets/img/image-d4-4.png)

9. さらに公式ドキュメントを読んでみたところ、global.cssの `@tailwind` がもっと最適化できるとのことだったのでとりあえずCSSおよびconfigファイルを書き換え

    [@GitHub]()

10. しかしこれでデプロイはできるもののスタイルは実際のサイトに反映されない・・・

    ![デプロイ後のマイページ：同じ状態](/assets/img/image-d4-2.png)

    諦めて一旦他の機能実装へ。


#### 3. 投稿機能をFix（Supabase連携）

1. 投稿機能UIはできているものの、投稿ロジックが繋がらずエラーが起き続けていたのでエラーを潰していく

    ![投稿エラー画面](/assets/img/image-d4-5.png)

    **テーブル名は基本的にcase sensitive**なので、コードとDBのテーブル名は完全一致である必要がある

    例：
    - ⭕️ Case Sensitive - Case Sensitive (完全一致)
    - ❌ Case Sensitive - Case sensitive
    - ❌ Case Sensitive - case sensitive
    = 要は大文字小文字が完全に区別されている

2. これでもエラーが解決しない
    ダミーデータとして入れた下記の項目の `id: "1"` が干渉している可能性があるので、 **新規投稿時はeditingPostを渡さないようにする** 必要がある

    [@GitHub: ]

    これによりダミーデータも使わなくなるので一時的にDisableする

    [@GitHub: Temp disable dummy data](https://github.com/kaaisz/lapsy-front/commit/f83e59e3edaadab01b165471605b5d943e7b59fa)

    デプロイすると、投稿を編集となっていた画面が切り替わっている！

    ![新規投稿画面](/assets/img/image-d4-7.png)

    投稿で「テストだよ〜」と打って保存すると、実画面には反映されないが、DBには連打した形跡が👼

    ![DB画面](/assets/img/image-d4-6.png)

    実画面にこの投稿を反映させたいので、次へ

## 2-4. 実画面に投稿内容を反映させる (Read)

### 1. `/app/components/Timeline.tsx` を作成

`/prototype/components/Timeline.tsx` を `/app/components/Timeline.tsx` にコピー

### 2. `mypage/page.tsx` でTimelineを使う

npm installで以下の項目を追加してエラーを潰していく
(日付と時刻をフォーマットするパッケージ)
```
npm install date-fns
```

日付の表示周囲でつまづくので型定義を丁寧にやっていく

**投稿データの取得や状態管理は親コンポーネント（例: mypage/page.tsx）で行い、Timelineは「受け取ったデータを表示するだけ」にするのがReactのベストプラクティス。**

なので、Timeline.tsx = 子コンポーネントではuseState, useEffectは使用せず、また、DBとの接続も実施しない。

[@GitHub: ]()
[@GitHub: ]()
[@GitHub: ]()

デプロイすると、ページ下部にDBに保存されている投稿が出現した！

![DBが反映された](/assets/img/image-d4-8.jpg)

### 3. 投稿詳細表示のページを追加

1. `/prototype/components/PostDetail.tsx` を `app/components/PostDetail.tsx` に移動

2. 関連する `alert.jsx` も `app/components/ui`に移動
この時点で足りていない `class-variance-authority`を`npm install`
    ```
    npm i -D class-variance-authority
    ```
3. 他のファイルを参考に渡すpostsデータを定義。あとで必要だけど今入れるとエラーになってしまうデータは適宜コメントアウトする

    [@GitHub: Migrate PostDetail and alert components to display post detail](https://github.com/kaaisz/lapsy-front/commit/6bbcf0e3db3ab0f8da05175550813de8a47aa915)

    [@GitHub: Fixed bugs by temp commenting out](https://github.com/kaaisz/lapsy-front/commit/4e888915a5f940548c7307dc5871f8e5c172e527)

4. デプロイ後、タイムラインに表示される投稿をクリックすると、投稿の詳細が表示される

    ![投稿の詳細](/assets/img/image-d4-9.png)

## 2-5. 編集機能 (Update) を追加

既存の以下のコンポーネントを編集もできるようにアップデートする

- `/prototype/components/PostComposer.tsx` … 投稿作成・編集フォーム
- `/prototype/components/PostDetail.tsx` … 投稿詳細

### 1. 編集状態の管理・内容の保存処理

編集対象の投稿を管理するstateを追加 - `mypage/page.tsx`を更新。

- `editingPost` を渡す直前で `postDate` などをstringに変換
- `handleUpdatePost`の引数型を柔軟にすることで投稿の新規作成と編集の両方に対応する
    
    [@GitHub: Add editing functionality for posts with handleUpdatePost method](https://github.com/kaaisz/lapsy-front/commit/1dff0a8b00c27f6d15467959c4f3cf7e7544ae77)


### 2. 編集ボタンから編集モードへ

`PostDetail` の `onEdit` で編集モードに切り替え

- `fetchPosts`は`useEffect`の外に切り出して定義

    [@GitHub: Refactor fetchPosts function to be defined outside of useEffect](https://github.com/kaaisz/lapsy-front/commit/7e22ecbb089c0e98ddb4e88eb7cc210feeaae82a)

### 3. 編集フォームの表示

`editingPost` がある場合は `PostComposer` を編集モードで表示

- `editingPost`を渡す直前で`string`型に変換
    ※`editingPost!` はnullでないことを保証するTypeScriptの記法です（`editingPost`がnullでない場合のみこの分岐に入るため安全）。
    [@GitHub: Refactor editingPost and onSave on mypage/page.tsxt](https://github.com/kaaisz/lapsy-front/commit/2341e6aca536ba517c7c3cfd1430d529ee487cad)

以上で、`editingPost` が渡された場合は編集モード、なければ新規投稿モードになる。

### 4. アプリ上で確認

[/mypage](https://lapsy-front.vercel.app/mypage)での初回表示時は新規投稿、編集時は「投稿を編集」となる

投稿を編集して保存すると、

![編集ページ](/assets/img/image-d4-10.png)
![編集ページその２](/assets/img/image-d4-11.jpg)

編集したデータが格納されている！

![編集後のTOPページ](/assets/img/image-d4-12.jpg)

テーブルにも最新のデータが格納されている
![編集後のDB](/assets/img/image-d4-13.png)

## 2-6. 削除機能 (Delete) を追加

### 1. 削除処理の関数を作成

`mypage/page.tsx`に削除用の関数を追加し、`PostDetail`コンポーネントの`onDelete`にこの関数を渡す

[@GitHub: Add delete functionality for posts in mypage/page.tsx with handleDeletePost method]()

この投稿一覧の「テスト連打したから消してくよ〜」を消しに行く

![削除前の一覧](/assets/img/image-d4-14.png)

![削除前の編集画面](/assets/img/image-d4-15.png)

### 3. PostDetail側で削除ボタンを押すと確認ダイアログが出る

🗑️ボタンを押すと下のようなアラートが表示
![削除確認画面](/assets/img/image-d4-16.png)

ボタンを押すと、削除が実行されて元のページへ

削除後は`fetchPosts()`で投稿一覧を再取得し、画面を最新状態に保っている

また、削除後は詳細画面や編集画面を閉じて、一覧画面に戻るようにしている（上記のsetSelectedPost(null)など）。

![削除完了](/assets/img/image-d4-17.png)

# 3. CSSの最適化 (再訪)

[先ほどの工程]()で最適化できなかったCSSの設定を行う

AIを通して調べてみたところ、そもそも `app/` のどこにも`global.css` をimportしていなかった😇

### やったこと

#### globals.cssの書き換えと再配置

- Next.jsでは、**`app/layout.tsx`でグローバルCSSをimportする必要がある**ので、以下のとおりコードを実行

    **rootファイルの、かつ一番先頭にimportされている必要がある**

    [@Github: Add global CSS import in layout.tsx](https://github.com/kaaisz/lapsy-front/commit/65027d1e0940d6eaf7a22422164460736d07d0a9)

    ```tsx
    import "../styles/globals.css";  // ← これを一番上に追加

    "use client"
    import useSession from "@/hooks/useSession"
    // ...他のimport
    ```

#### コンパイラを通すための設定

- `postcss.config.mjs`と`postcss.config.js`が競合していたので、**`postcss.config.js`のみを残す**
    postcss.config.jsのコードは以下リンクのとおり

    [@Github: Remove unnecessary lines on postcss.config.js](https://github.com/kaaisz/lapsy-front/commit/24b7266c1792a52adf91e771d40b3d81ce7ba72e#diff-05ed1c7f99e45b485708115502a1e0f28c8547e9a9a29c1b8664f79103cf7873)

- tailwind v4.x系の仕様に沿い、`@tailwindcss/postcss`を念の為再度インストールし、`postcss.config.js`の内容も書き換える

    ```
    npm install @tailwindcss/postcss --save-dev
    ```

    ```js
    // postcss.config.js
    module.exports = {
    plugins: {
        '@tailwindcss/postcss': {},
        autoprefixer: {},
    },
    };
    ```

- 依存パッケージも書きコマンドを使って一度消去し、再度インストール
    ```
    rm -rf node_modules package-lock.json
    npm install
    npm run build
    ```

これで、`npm run build`をしたときにcssがコンパイルされるようになる

しかしコンパイルが通っても実サイトに反映されない

#### 実サイトに反映させるためのエラー潰し

ここからひたすらエラーとの戦いになる。CSSのコンパイルが通っているかはlocalでも確認できる

1. `npm run dev`し、`http://localhost:3000` にアクセスすると、ひたすらエラーが出てハマりはじめる

2. Next.jsのapp/layout.tsxでは<html>と<body>タグが必須になったため、`app/layout.tsx`は以下のように修正

    [@GitHub: app/layout.tsx](https://github.com/kaaisz/lapsy-front/commit/0b35b607393d8e8d902f3b724bf6f4b13f0dc855#diff-eca96d2c09f31517696a26e1d0be4070e1fbab02831481bed006e275741d030b)

    - ↑ Next.js（App Router構成）では、app/layout.tsxの返り値が必ず`<html>`と`<body>`タグでラップされている必要がある
    - app/layout.tsxのRootLayoutが「クライアントコンポーネント （`"use client"`）」になっている場合に、`<html>`や`<body>`タグを返してはいけない
    - =  **app/layout.tsxは　`<html>`や`<body>` と"use client"を一緒に付けてはいけない。** つけた場合はエラーになる

#### `/app/components/AuthGuard.tsx`の設置

layout.tsxを書き換えたので、クライアントサイドの認証やリダイレクト処理を行う子コンポーネント `AuthGuard.tsx`を作成

このとき、**app/layout.tsx に書いてあるクライアント専用フックやロジックのimportは全て削除**

<aside>

### ⚠️ app/layout.tsxはサーバーコンポーネント

- クライアント専用フック (`useSession`や`usePathname`、`useRouter`、`useEffect`) や `"use client"` つきのimportは絶対にしないこと。
- 代わりに`AuthGuard`を`"use client"`付きのクライアントコンポーネントとして実装する。
- 認証ガードは各ページ（`app/page.tsx`や`app/mypage/page.tsx`など）でラップするが、**`app/layout.tsx`では絶対にクライアントコンポーネントでラップしない = AuthGuardを使わない**

</aside>

```tsx
//app/components/AuthGuard.tsx

"use client";
import useSession from "@/hooks/useSession";
import { usePathname, useRouter } from "next/navigation";
import { useEffect } from "react";

export default function AuthGuard({ children }: { children: React.ReactNode }) {
  const { session, loading } = useSession();
  const pathname = usePathname();
  const router = useRouter();

  useEffect(() => {
    if (!loading && !session && pathname !== "/login" && pathname !== "/register") {
      router.push("/login");
    }
    if (!loading && session && (pathname === "/login" || pathname === "/register")) {
      router.push("/mypage");
    }
  }, [session, loading, pathname, router]);

  if (loading) return <p>Loading...</p>;
  if (!session && pathname !== "/login" && pathname !== "/register") return null;

  return <>{children}</>;
}
```

これでようやくCSSの反映が確認できた〜🥳
![ダッシュボード](/assets/img/image-d4-18.png)

残りのCSSの問題については可能な限り潰しつつ、最低条件になっているEC2との同期まで終わらせたいのでそちらを優先的に実行


# 最終的にVercelとEC2をどう繋げるか？

ChatGPT曰く、以下のような回答だった。実務でも使える経験にしたいのでなるべく一般的かつ合理的な手段を選びたいので今後の参考とする。

> - 開発はVercelで快適に続けたいが、最終的な本番環境はEC2に置きたい、は **可能、かつ王道的**
> - 現実的なおすすめルートは、
>   1. Vercelで開発・デプロイ（今の構成）
>   2. 完成したら `next build` → `next export` or `next start` を EC2 に移行
>  - EC2でのNext.js動作方法 → `next start`（Node.jsアプリとして運用）※速さ重視なら静的サイトベースの `next export` でもOK
> - デプロイ手段 → GitHub Actions による SSH デプロイ（CI/CD）

# EC2に繋げたい場合の最短経路

EC2に現在のプロジェクトを同期させる最短手順と、一般的な所要時間の目安

## 最短手順(ダイジェスト)

### 1. EC2にSSHで接続
```sh
ssh -i <秘密鍵ファイル> ec2-user@<EC2のパブリックIP>
```
- `<秘密鍵ファイル>`：EC2作成時にダウンロードした.pemファイル
- `<EC2のパブリックIP>`：AWS管理画面で確認

### 2. EC2にNode.js・npm・gitをインストール（未インストールの場合）
```sh
sudo yum update -y
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs git
```

### 3. ローカルからEC2へプロジェクトを転送
#### 方法A: scpコマンドで直接転送（最短）
ローカルのターミナルで、プロジェクトディレクトリの1つ上の階層で実行：
```sh
scp -i <秘密鍵ファイル> -r lapsy-front ec2-user@<EC2のパブリックIP>:~/
```
#### 方法B: GitHub等のリポジトリを使う場合
1. プロジェクトをGitHub等にpush
2. EC2でclone
   ```sh
   git clone <リポジトリURL>
   ```

### 4. EC2で依存パッケージをインストール
```sh
cd ~/lapsy-front
npm install
```

### 5. 必要に応じて環境変数や設定ファイルを配置

### 6. サーバー起動（例: Next.js開発サーバー）
```sh
npm run dev
```
または本番用
```sh
npm run build
npm start
```

## 所要時間の目安

- **SSH接続**：数秒
- **Node.js/gitインストール**：1～3分
- **プロジェクト転送（scp）**：  
  - 100MB未満なら1～2分（回線速度次第）
- **npm install**：1～3分（依存数・回線次第）
- **サーバー起動**：数秒～1分

**合計：約5～10分程度**（ネットワークやEC2の性能により前後）

### 注意
- EC2のセキュリティグループで22番ポート（SSH）や必要なWebポート（80, 3000等）が開いているか確認してください。
- `.env`などの機密ファイルはscpや手動で個別転送してください。


# Express APIとの連携 (JWT付き)

<aside>

### 💡 なぜこのプロセスが必要なのか？

主には、 **フロントエンド（Next.js + Supabase）から、EC2上のExpress APIにセキュアにアクセスする** ため

- **Supabaseでログインしていても、Expressサーバーにはセッション情報は共有されない** = 何も対策しなければ誰でもExpressのAPIを叩けてしまう

- Supabaseが発行するJWT（アクセストークン）をヘッダーに付けてリクエスト → Express側でそのJWTを検証し、「これは本当にログイン済みユーザーだ」と判定する

= 「ログイン済ユーザーだけが安全にAPIを使える状態」
を構築するための **本番運用では必須レベルの認証接続手順**

</aside>




