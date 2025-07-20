<link href="/assets/css/style.css" rel="stylesheet" />

[←TOPへ戻る](document.html)

# 自律的情報技術学習演習：作業の記録 Day-3〜後半日程

DS233292 鈴木可愛

## これは何

ムサビ通信「自律的情報技術学習演習」の3日目の授業の記録です(Day3と記載していますが[前回](document-d2.html)の続き〜後半日程にかけての記録です)

前回はセットアップ〜ログイン周りが終わったので、ログイン後ページ以降の整理を行います。この辺から既存UIとの融合にCursor, Claude Codeも使用しつつ開発を進めます

※一部ドキュメント内容を割愛したり後ほど更新予定

<aside>

#### 🔖 目次

- [作成中のフロントエンドURL](#anchor1)
- [Figma Make製プロトタイプURL](#anchor2)
- [ところで：デザイン済みのUIの実装はどのタイミングでやればいいの](#anchor3)
- [今日やること](#anchor4)
- [1. ログイン状態の保持(セッション管理)](#anchor5)
  - [1-1. useSession を作る（共通化）](#anchor5-1)
  - [1-2. 1-1. をlayout.tsxとpage.tsxで使う](#anchor5-2)
    - [ところで：.tsと.tsxの違い](#anchor5-2-1)
  - [1-3. セッションに基づいたUI制御を各所で使う準備](#anchor5-3)
  - [1-4. セッションによりルーティング制御を加える](#anchor5-4)
    - [ここまでのファイル構成の確認](#anchor5-4-1)
    - [create-next-app の実行時からあるもので不要なもの](#anchor5-4-2)
- [💬 Cursorと話し始める](#anchor6)
  - [1-5. Layout.tsxを追加、mypage/、login/のpage.tsxも修正](#anchor5-5)
  - [1-6. 内容を確認する](#anchor5-6)
- [2. 新規登録画面の作成](#anchor6)
  - [2-1. Supabase上で profiles テーブルを作成](#anchor7-1)
    - [💡 Supabaseあれこれ](#anchor7-1-1)
  - [2-2. register/page.tsx を作成](#anchor7-2)
  - [2-3. 新規登録できるか確認](#anchor7-3)
    - [RSLを無効化](#anchor7-3-1)
    - [SupabaseのSite URLをVercelのURLに変更](#anchor7-3-2)

</aside>




<a id="anchor1"></a>

# 作成中のフロントエンドURL

[https://lapsy-front.vercel.app/](https://lapsy-front.vercel.app/)



<a id="anchor2"></a>

# Figma Make製プロトタイプURL

[https://flow-swan-91912375.figma.site/](https://flow-swan-91912375.figma.site/)

<a id="anchor3"></a>

# ところで：デザイン済みのUIの実装はどのタイミングでやればいいの

ChatGPTによると、ログインログアウトができるようになったタイミングが最も早いらしい

ログインはできているので、先にログアウト機能の実装を済ませてからUIの調整に入る

<a id="anchor4"></a>

# 今日やること

1. ログイン状態の保持(セッション管理)
2. ログアウト機能の実装
3. 認証ガード付きページ = マイページの作成
4. Supabase DBへのユーザー情報保存
5. Express APIとの連携（JWT付き）
6. UI実装 - Figma Makeとの統合
    1. Dashboard.tsx を認証済ユーザー専用ページに統合
    2. ActionCards.tsx に Supabase DBの内容をマッピング
    3. App.tsx を Next.js に移植する（エントリーポイント調整）
    4. 優先度4：Claude Codeでの自然言語統合支援（任意で随時）
    5. 優先度5：デザインテーマ・UIスタイルの維持（Tailwind含む）

<a id="anchor5"></a>

# 1. ログイン状態の保持(セッション管理)
### 〜`supabase.auth.getSession()` を使ってログイン状態を確認〜

ログイン判定後は、その状態保持も必要

これによりダッシュボード機能の土台が出来上がる

<a id="anchor5-1"></a>

## 1-1. `useSession` を作る（共通化）

```tsx
// hooks/useSession.ts
"use client"
import { useState, useEffect } from "react"
import { supabase } from "@/lib/supabaseClient"

export default function useSession() {
  const [session, setSession] = useState(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // 初回読み込みでセッション取得
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session)
      setLoading(false)
    })

    // ログイン/ログアウト時の監視
    const { data: listener } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session)
    })

    return () => {
      listener?.subscription?.unsubscribe()
    }
  }, [])

  return { session, loading }
}

```

<a id="anchor5-2"></a>

## 1-2. 1-1. をlayout.tsxとpage.tsxで使う

<aside>

💣 後述手順6に記載した通り、 **layout.tsxとpage.tsxは両方必要** だった。この手順に記載の内容はログ用なので以下の内容は無視して3.へ進む

</aside>


特定ページでの使用も可能。「ログイン状態をどこで保持しておきたいか」の目的に応じてどちらかを作成すればOK

1. 全ページに対して「ログインしていなければ全体を非表示にする」など共通ルールを適用したいとき → `layout.tsx` 
    - 全ページ共通のレイアウトやラップ処理を行うファイル
    - たとえばライフログアプリで、ログインなしでは何も使えないようにしたい時
2. 特定のページだけに認証ガードを付けたい → `page.tsx` 
   - そのページ専用の表示ロジックを書くファイル
   - たとえばTOPページやAboutは誰でも見られるが、マイページだけ制限したい時

このロジックはあとで変更も可能

今回はTOPページ配下にLPを作成する可能性なども考慮して、**2でとりあえず進行**

```tsx
// app/layout.tsx または app/page.tsx どちらでも使用可能
"use client"
import useSession from "@/hooks/useSession"

export default function Layout({ children }: { children: React.ReactNode }) {
  const { session, loading } = useSession()

  if (loading) return <p>Loading...</p>

  return (
    <>
      {/* ログインしているときだけUI表示 */}
      {session ? children : <p>ログインしてください</p>}
    </>
  )
}

```
<br>

<a id="anchor5-2-1"></a>

<aside>

💡 **ところで：.tsと.tsxの違い**

- .ts
    - 純粋なTypeScriptファイル
    - JSX要素の追加をサポートしない
- .tsx
    - JSXを含むファイル
    - 型アサーションの記法として value as type と <type>value の2通りあるが、後者は .tsx には書けない
（ <> はJSXタグのマーカーであるため）

Reactを使うプロジェクト内のTypeScriptファイルにおいて、`.ts` と `.tsx` は明示的に分けるべき。<br>拡張子で明示的にしておくことで、「このファイルにJSXを書くべきではない」こと = UIコンポーネントのファイルか、ロジックを書くファイルか、を表すことができる。

</aside>

<a id="anchor5-3"></a>

## 1-3. セッションに基づいたUI制御を各所で使う準備

（例：ログアウトボタンの表示制御）

```tsx
// components/Header.tsx
"use client"
import useSession from "@/hooks/useSession"

export default function Header() {
  const { session } = useSession()

  return (
    <header>
      {session ? (
        <p>{session.user.email} さん、こんにちは</p>
      ) : (
        <p>ゲストユーザー</p>
      )}
    </header>
  )
}

```

<a id="anchor5-4"></a>

## 1-4. セッションによりルーティング制御を加える

今回は *2. 1. をLayoutで使う* で決定した通り、mypage/page.tsxなどで、未ログインなら/loginへ強制リダイレクトさせたいのでこちらの手順も実施

```tsx
"use client"
import { useRouter } from "next/navigation"
import useSession from "@/hooks/useSession"
import { useEffect } from "react"

export default function MyPage() {
  const { session, loading } = useSession()
  const router = useRouter()

  useEffect(() => {
    if (!loading && !session) {
      router.push("/login")
    }
  }, [loading, session])

  if (loading || !session) return <p>読み込み中...</p>

  return <div>これはログイン済みユーザーだけが見れるページです</div>
}

```

<a id="anchor5-4-1"></a>

### ここまでのファイル構成の確認

以下のようなファイル構成になっている

```vbnet
my-app/
├── app/
│   ├── layout.tsx               ← 全体レイアウト（必要に応じて useSession 使用）
│   ├── page.tsx                 ← TOPページ（公開 or 認証あり）
│   ├── login/
│   │   └── page.tsx             ← ログイン画面
│   ├── mypage/
│   │   └── page.tsx             ← 認証済ユーザー専用ページ（ガード付き）
│
├── components/
│   ├── Header.tsx              ← ヘッダーコンポーネント（セッション状態で表示変更）
│   ├── Footer.tsx              ← フッターなど
│
├── hooks/
│   └── useSession.ts           ← カスタムフック（セッション管理を共通化）
│
├── lib/
│   └── supabaseClient.ts       ← Supabaseクライアント初期化
│
├── styles/
│   └── globals.css             ← TailwindやグローバルCSS
│
├── public/
│   └── favicon.ico             ← 公開ファイル
│
├── tailwind.config.js         ← Tailwind設定
├── tsconfig.json              ← TypeScript設定
└── next.config.js             ← Next.js設定

```

<br>

<a id="anchor5-4-2"></a>

### `create-next-app` の実行時からあるもので不要なもの

| ファイル/フォルダ                          | 削除してよい理由                                          |
| ---------------------------------- | ------------------------------------------------- |
| `pages/` フォルダ全体                    | App Router（`app/`）ベースで開発しているため不要                  |
| `pages/index.tsx`                  | `app/page.tsx` に置き換わっている                          |
| `pages/_app.tsx`, `_document.tsx`  | `app/layout.tsx` を使う構成なので不要                       |
| `public/vercel.svg`                | Vercelの初期デモ画像。UIに使っていなければ不要                       |

<a id="anchor6"></a>

# 💬 Cursorと話し始める

実はここまでずっとブラウザのChatGPTとCursorを行ったり来たりしていたが何度やってもデプロイがうまくいかずハマり始めたのでCursorで `GPT-4.1` との対話に変更。最初からこうした方が早かった。早く気づけ自分。

<a id="anchor5-5"></a>

## 1-5. Layout.tsxを追加、mypage/、login/のpage.tsxも修正

GPT-4.0曰く、

> Next.jsのApp Router構成ではapp/page.tsxはトップページ用であり、共通レイアウトにはapp/layout.tsxを使うのが一般的です。
layout.tsxが存在しないため、各ページで個別に認証ガードを実装している状態です。
> - 共通の認証ガードがないため、各ページで重複した認証チェックをしている。
> - app/layout.tsxが存在しないため、全体のルーティング制御やUI共通化ができていない。
> - app/page.tsxの内容はトップページ専用であり、共通レイアウトには適していません。

とのことなので、layout.tsxを新規追加

```tsx
//layout.tsx

"use client"
import useSession from "@/hooks/useSession"
import { usePathname, useRouter } from "next/navigation"
import { useEffect } from "react"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const { session, loading } = useSession()
  const pathname = usePathname()
  const router = useRouter()

  useEffect(() => {
    if (!loading && !session && pathname !== "/login") {
      router.push("/login")
    }
    if (!loading && session && pathname === "/login") {
      router.push("/mypage")
    }
  }, [session, loading, pathname, router])

  if (loading) return <p>Loading...</p>
  if (!session && pathname !== "/login") return null

  return <>{children}</>
} 
```

<br>

mypage/page.tsx、login/page.tsx、components/Header.tsx、useSession.tsも修正。

Session状態のハンドリングのためにtsおよびtsxではSession方定義が必要だったので、関連するファイルの型を全て追加・修正する必要があった

以下、このまま貼ってOKなコード

```tsx
// login/page.tsx
"use client"

import { useState } from "react"
import { useRouter } from "next/navigation"
import { supabase } from "@/lib/supabaseClient"

export default function LoginPage() {
  const router = useRouter()
  const [email, setEmail] = useState("")
  const [password, setPassword] = useState("")
  const [error, setError] = useState("")
  const [loading, setLoading] = useState(false)
  
  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError("")

    const { error } = await supabase.auth.signInWithPassword({
      email,
      password,
    })

    if (error) {
      setError(error.message)
    } else {
      router.push("/mypage") // ログイン成功後にマイページへ
    }

    setLoading(false)
  }

  return (
    <div className="max-w-md mx-auto mt-10 p-4 border rounded">
      <h1 className="text-xl font-bold mb-4">ログイン</h1>
      <form onSubmit={handleLogin} className="space-y-4">
        <div>
          <label className="block mb-1">メールアドレス</label>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="w-full border px-3 py-2 rounded"
            required
          />
        </div>
        <div>
          <label className="block mb-1">パスワード</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="w-full border px-3 py-2 rounded"
            required
          />
        </div>
        {error && <p className="text-red-500">{error}</p>}
        <button
          type="submit"
          className="w-full bg-black text-white py-2 rounded disabled:opacity-50"
          disabled={loading}
        >
          {loading ? "ログイン中..." : "ログイン"}
        </button>
      </form>
    </div>
  )
}
```

<br>

```tsx
// mypage.tsx
// 認証済ユーザー専用ページ
"use client"

import useSession from "@/hooks/useSession"
import type { Session } from "@supabase/supabase-js"

export default function MyPage() {
  const { session, loading }: { session: Session | null, loading: boolean } = useSession() as { session: Session | null, loading: boolean }

  if (loading) return <p>読み込み中...</p>
  if (!session) return null // layout.tsxでリダイレクトされる

  return (
    <div>
      <h1>こんにちは、{session.user.email} さん！</h1>
      <p>これはログイン済ユーザー専用ページです。</p>
    </div>
  )
}
```

<br>

```tsx
// components/Header.tsx
"use client"
import useSession from "@/hooks/useSession"
import type { Session } from "@supabase/supabase-js"

export default function Header() {
  const { session }: { session: Session | null } = useSession() as { session: Session | null }

  return (
    <header>
      {session ? (
        <p>{session.user.email} さん、こんにちは</p>
      ) : (
        <p>ゲストユーザー</p>
      )}
    </header>
  )
}
```

<br>

```tsx
// hooks/useSession.ts
// セッション管理用のカスタムフック - Supabaseのセッションを取得して共通管理
"use client"
import { useState, useEffect } from "react"
import { supabase } from "@/lib/supabaseClient"
import type { Session } from "@supabase/supabase-js"

export default function useSession() {
  const [session, setSession] = useState<Session | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    // 初回読み込みでセッション取得
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session)
      setLoading(false)
    })

    // ログイン/ログアウト時の監視
    const { data: listener } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session)
    })

    return () => {
      listener?.subscription?.unsubscribe()
    }
  }, [])

  return { session, loading }
}
```

<br>

`app/page.tsx` も「ページコンポーネント」ではなく「レイアウトコンポーネント」として書かれているためエラーになってしまい、修正が必要だった
Next.jsのapp/page.tsxは「ページ」なので、
propsでchildrenを受け取ってはいけない（childrenはレイアウト専用）
単純なページコンポーネント（export default function Page() { ... }）である必要がある

```tsx
// app/layout.tsx または app/page.tsx どちらでも使用可能
"use client"
import useSession from "@/hooks/useSession"
import type { Session } from "@supabase/supabase-js"

export default function Page() {
  const { session, loading }: { session: Session | null, loading: boolean } = useSession() as { session: Session | null, loading: boolean }

  if (loading) return <p>Loading...</p>
  if (!session) return <p>ログインしてください</p>

  return <p>ようこそ、{session.user.email} さん！</p>
}
```

<a id="anchor5-6"></a>

## 1-6. 内容を確認する

![ログイン画面](/assets/img/image-d3-1.png)

実装に成功！・・・ただしログインできる情報がない

<br>

<a id="anchor7"></a>

# 2. 新規登録画面の作成

<a id="anchor7-1"></a>

## 2-1. Supabase上で `profiles` テーブルを作成

1. Dashboardから自分のプロジェクトに入り、「Get started by building out your database」のセクション下のボタンから遷移するか、
![Dashboardの「Get started by building out your database」のセクション](/assets/img/image-d3-2.png)
サイドバーを開くと出てくる
![alt text](/assets/img/image-d3-3.png)
「Table Editor」「SQL Editor」のいずれかから選択。
2. Table Editor(GUI)を選択した場合
    遷移先で出現する下記画像のボタンをクリック
    ![Create Tableボタン](/assets/img/image-d3-5.png)
    以下の項目を追加
    📄 Name = テーブル名 = `profile`
    ![Name](/assets/img/image-d3-6.png)
    ![テーブル追加項目](/assets/img/image-d3-4.png)
    以下の画面のように追加して、saveを押す
    ![Create Table](/assets/img/image-d3-7.png)
3. SQL Editorを選択した場合
以下の項目を追加
    ```sql
    // SQL
    create table profiles (
    id uuid primary key references auth.users(id),
    email text,
    created_at timestamp with time zone default timezone('utc'::text, now())
    // 任意でusername, avatar_url なども追加可能
    );
    ```

<a id="anchor7-1-1"></a>

<aside>

### 💡 Supabaseあれこれ

#### 1. SupabaseはpostgreSQLベース

- あとから柔軟にカラム（項目）を追加・変更可能
- MySQLよりPostgreSQLは多くの機能を備え、複雑なクエリやデータ処理、大規模データセットにも最適（MySQLはシンプルで高速な読み取り操作が可能 = 小規模〜中規模アプリケーション向け）
- ドキュメントもMySQLに比較して数が多め
- 参考リンク：[PostgreSQL vs MySQL: その違いとは](https://qiita.com/Integrateio/items/f1e1e20fd96edbfe3412)

#### 2. Supabaseではユーザー管理に `profiles` テーブル名の使用が推奨される

- **`users` はSupabase内部で予約済のシステムテーブル** - Supabaseでは、認証機能を有効にすると自動的に `auth.users` というシステムテーブルが作成されます。
    - これは 認証（email, password, uid など）を管理するための内部テーブル
    - Supabaseの他の機能（セッション、ログイン状態、API）でも使用されます。
- なお、他のBaaSやORMでも同様の傾向がある

#### 3. DB構造変更時の注意点

- text → integer のような変更は型エラーになることがあるので、型変更には注意
- 削除前に、フロントエンドやAPIでそのカラムに依存していないか、その使用箇所を確認
- 後から `UNIQUE` や `FOREIGN KEY` をつける場合は注意が必要
- 大規模な変更の前には export（CSV）などでバックアップを

</aside>

<a id="anchor7-2"></a>

## 2-2. register/page.tsx を作成

サインアップ成功直後に `profiles` テーブルへ挿入する

※Supabaseで作成したDBに最適化済

```tsx
// 新規登録画面
"use client"

import { useState } from "react"
import { useRouter } from "next/navigation"
import { supabase } from "@/lib/supabaseClient"

export default function RegisterPage() {
  const router = useRouter()
  const [email, setEmail] = useState("")
  const [password, setPassword] = useState("")
  const [error, setError] = useState("")
  const [loading, setLoading] = useState(false)

  const handleRegister = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError("")

    const { data: signUpData, error: signUpError } = await supabase.auth.signUp({
      email,
      password,
    })

    if (signUpError) {
      setError(signUpError.message)
      setLoading(false)
      return
    }

    // サインアップ後にprofilesテーブルへ登録
    const user = signUpData.user
    if (user) {
      const { error: insertError } = await supabase
        .from("profiles")
        .insert([
          {
            id: user.id,
            email: user.email,
          },
        ])
      if (insertError) {
        setError("プロフィール作成に失敗しました")
        setLoading(false)
        return
      }
    }

    router.push("/mypage") // 登録成功後にマイページへ遷移
    setLoading(false)
  }

  return (
    <div className="max-w-md mx-auto mt-10 p-4 border rounded">
      <h1 className="text-xl font-bold mb-4">新規登録</h1>
      <form onSubmit={handleRegister} className="space-y-4">
        <div>
          <label className="block mb-1">メールアドレス</label>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="w-full border px-3 py-2 rounded"
            required
          />
        </div>
        <div>
          <label className="block mb-1">パスワード</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="w-full border px-3 py-2 rounded"
            required
          />
        </div>
        {error && <p className="text-red-500">{error}</p>}
        <button
          type="submit"
          className="w-full bg-black text-white py-2 rounded disabled:opacity-50"
          disabled={loading}
        >
          {loading ? "登録中..." : "新規登録"}
        </button>
      </form>
    </div>
  )
} 
```

<a id="anchor7-3"></a>

## 2-3. 新規登録できるか確認

[https://lapsy-front.vercel.app/register](https://lapsy-front.vercel.app/register)

![新規登録画面](/assets/img/image-d3-8.jpg)

新規登録画面には行けるがプロフィール作成に失敗してしまった

原因はSupabaseとの連携ミスによるものらしいが、該当箇所を直してもエラーが解決しないので、サインアップタイミングでエラーハンドリングコードを撒いて様子を見る

```tsx
    // const [loading, setLoading] = useState(false) の下に、下にreturnが来るように続ける

    const handleRegister = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError("")

    const { data: signUpData, error: signUpError } = await supabase.auth.signUp({
      email,
      password,
    })

    if (signUpError) {
      setError(signUpError.message)
      setLoading(false)
      return
    }

    // サインアップ後にprofilesテーブルへ登録
    const user = signUpData.user
    if (user) {
      const { error: insertError } = await supabase
        .from("profiles")
        .insert([
          {
            id: user.id,
            email: user.email,
          },
        ])
      if (insertError) {
        setError("プロフィール作成に失敗しました: " + insertError.message)
        console.error(insertError)
        setLoading(false)
        return
      }
    } else {
      setError("ユーザー情報が取得できませんでした。メール認証が必要な場合はメールを確認してください。")
      setLoading(false)
      return
    }

    router.push("/mypage") // 登録成功後にマイページへ遷移
    setLoading(false)
  }
```

エラー画面

![エラー画面](/assets/img/image-d3-9.jpg)

これは**テーブル作成時にRLS(Row Level Security)が有効になっていた**ことが原因らしい。(エラーに `new row violates row-level security policy for table "profiles` と表示されたことから判断) = Supabaseのダッシュボードで「Add RLS Policy」と表示されていても、RLS自体は有効化されており、**明示的に「挿入（insert）」を許可するポリシーを作成しないと、どのユーザーも書き込みできない。**

<a id="anchor7-3-1"></a>

### RSLを無効化

以下SQLをSupabaseのSQLエディタで実行し、RSLを無効化
1. RLSを一時的に無効化する（開発用）
    ```SQL
    alter table profiles disable row level security;
    ```

2. RLSを有効のまま「認証ユーザーのinsertを許可」するポリシーを追加
    ```SQL
    -- 認証ユーザーが自分自身のプロフィールをinsertできるようにする
    create policy "Allow authenticated user to insert their profile"
    on profiles
    for insert
    with check (auth.uid() = id);
    ```

これで再度新規登録にトライすると、、、

![Table Editorで新規登録成功が確認できる](/assets/img/image-d3-10.jpg)

登録ができた！SupabaseのTable Editorでレコードの追加が確認できる

しかし、ログインを試みると、`Invalid login credentials` と言われてマイページに遷移ができない。

SupabaseはAuthのためにメールを送ってくれるが、AuthのURLがlocalhostだった

<a id="anchor7-3-2"></a>

### SupabaseのSite URLをVercelのURLに変更

1. 左メニュー「Authentication」→「URL Configuration」または「Settings」→「Auth Settings」
2. 「Site URL」欄をvercelのURLに変更して保存

もう一度新規登録からやり直すため、一度テーブルからデータを削除して再トライ。

サインアップ → ログイン後に「Invalid login credentials」が発生したが、Supabase側からパスワードの再発行をし、送られてきたメールリンクをクリックしたところログインに成功

![TOPページ](/assets/img/image-d3-12.png)
TOPページのヘッダー

![マイページトップ](/assets/img/image-d3-11.png)
マイページトップ


パスワード保存の不整合や、メール認証・セッションのタイミングによる一時的な不整合が原因だった可能性が高いらしい。

本番運用時にサインアップ・認証・ログインの一連の流れを複数回テストし、ユーザー体験を確認する必要がありそう



# 3. ログイン体験の最適化

ここまでのエラーハンドリングも鑑みて、ログイン時の誘導などに関する最適化を実施

※これ以降、既存コードの差分が増えるので詳細はGitHubリンク参照

[@GitHub: Enhance registration feedback by adding informational messages for users upon successful registration](https://github.com/kaaisz/lapsy-front/commit/ffc14d5431f6651060f66efe912b130c4ecb6212)



# 4. ログアウト機能も実装する

ログインができてログアウトができない魔球のようなアプリはUXも損ねるので、ログアウト機能も作成する

とりあえず、ログアウトボタンを `Header.tsx` に追加

[@GitHub: Add logout functionality to Header component, allowing users to sign out and redirect to login page](https://github.com/kaaisz/lapsy-front/commit/846d71fa4f23baf27515e9dcf2a694e8952c5580)

![マイページトップ](/assets/img/image-d3-11.png)
しかしこのままだと `/mypage` にHeaderが出現していない

ので、`/mypage/page.tsx` に `<Header />` を追加

[@GitHub: Add Header component to MyPage for improved user interface](https://github.com/kaaisz/lapsy-front/commit/d24bafa8961934877c843515b57a72f879236610)

[@GitHub: Wrap component and elements to fix the error](https://github.com/kaaisz/lapsy-front/commit/4ede8d8aa0a4c07bce91f4d3d71ca65c5c9bc6df)

![ログアウトボタンの実装](/assets/img/image-d3-13.png)

ログアウトボタンが `/mypage` に実装され、ボタンを押すと `login` に遷移するようになった

長くなってきたので[Day4](document-d4.html)に続く... →