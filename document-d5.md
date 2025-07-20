<link href="/assets/css/style.css" rel="stylesheet" />

[←TOPへ戻る](document.html)

# 自律的情報技術学習演習：作業の記録 Day-5(ほぼ最終日)

DS233292 鈴木可愛

## これは何

ムサビ通信「自律的情報技術学習演習」の3日目以降〜後半日程の授業の記録です
CRUD処理ができたので発展的内容やプロトタイプとの融合を進めつつ、いよいよEC2に上げる準備を進めていきます

# 今日やること：

最低限の動くものとしての実装はできているので、エラーを潰しつつ側だけでもいけるものはそれだけで済ませ、最小構成でもいいので本番にアップしようという想定

1. EC2環境の設定 (優先度 ★★★★★)
2. CSS最適化・プロトタイプからのコンポーネントの移動 (優先度 ★★★★)
3. GitHub Actionsなどでの自動化・JWTによるセキュア化の設定(優先度 ★★★★)
4. 下書き機能の実装・カレンダーとの同期などの発展的内容の動的実装 (優先度 ★★)

# 1. CSS最適化・コンポーネントの移動

Tailwindをコンパイル含めしっかり触りたかったが、うまくいかず時間もなかったのでプロトタイプURLから拾えるCSSを丸ごとglobals.cssにコピペ

[@GitHub:Temp replace globals.css to check how the actual code in proto looks like](https://github.com/kaaisz/lapsy-front/commit/72f5e447f5f578f6af883307ebfcd6ce18395eab#diff-4ee0e71a2145586038d583d58bcbd7aaa6b42318f417f5b62f2cbee48931861b)

残りのコンポーネントも移動し、とりあえず側だけ完成したので後で細かなロジック含めて調整

# プロジェクト名を変更する

リポジトリとVercelプロジェクト名を`lapsy-front`から`lapsy`に非破壊的に変更する

## 1. GitHubリポジトリ名の変更

### ブラウザでGitHubにアクセス
1. GitHubの該当リポジトリページにアクセス
2. **Settings** タブをクリック
3. **General** セクションの一番下にある **Repository name** を見つける
4. `lapsy-front` を `lapsy` に変更
5. **Rename** ボタンをクリック

## 2. ローカルリポジトリのリモートURL更新

```bash
# 現在のリモートURLを確認
git remote -v

# 新しいURLに更新（ユーザー名は適宜変更）
git remote set-url origin https://github.com/YOUR_USERNAME/lapsy.git

# 更新確認
git remote -v
```

## 3. Vercelプロジェクト名の変更

### Vercel ダッシュボードで変更
1. [Vercel Dashboard](https://vercel.com/dashboard) にアクセス
2. `lapsy-front` プロジェクトをクリック
3. **Settings** タブをクリック
4. **General** セクションの **Project Name** を `lapsy` に変更
5. **Save** をクリック

### または、Vercel CLIを使用
```bash
# Vercel CLIをインストール（未インストールの場合）
npm i -g vercel

# プロジェクト設定を更新
vercel --prod
# プロンプトで新しいプロジェクト名 'lapsy' を入力
```

<aside>

☝️ Vercelでプロジェクト名を変更する際、このようなcautionが出ることがあるが、この段階ではまだCI/CDを使用していないので今回は気にしなくてOK
```
Changing the project name will affect the OpenID Connect Token claims and may require updating your backend's OpenID Connect Federation configuration. Consult the OIDC documentation for more information.
```

安全に変更したい場合の手順

1. .env.localを確認する
    ```bash
    # .env.local の確認（Vercel関連の環境変数があるか）
    cat .env.local 2>/dev/null | grep -i vercel
    ```
2. バックアップを作成してからデプロイテスト
    ```bash
    # 1. バックアップ作成
    git add .
    git commit -m "backup before name change"

    # 2. プロジェクト名変更
    # （前述の手順でVercelダッシュボードで変更）

    # 3. デプロイテスト
    vercel --prod
    ```

3. 念のため以下も確認すると安心
- ログイン/ログアウト機能のテスト
- 環境変数の動作確認
- Supabaseとの接続確認

</aside>

## 4. 設定ファイルの更新

### package.json の更新
```bash
# package.jsonのname フィールドを確認・更新
```

package.json
```json
"name": "lapsy", // 変更
"version": "0.1.0",
"private": true,
```


# 2. EC2環境の設定

## EC2に繋げたい場合の最短経路

EC2に現在のプロジェクトを同期させる最短手順と、一般的な所要時間の目安

### 最短手順(ダイジェスト: ubuntuの場合)

#### 1. EC2にSSHで接続
```sh
ssh -i <秘密鍵ファイル> ec2-user@<EC2のパブリックIP>
```
- `<秘密鍵ファイル>`：EC2作成時にダウンロードした.pemファイル
- `<EC2のパブリックIP>`：AWS管理画面で確認

#### 2. EC2にNode.js・npm・gitがインストールされているか確認

```
# Node.jsの確認
node --version

# npmの確認
npm --version

# gitの確認
git --version
```

#### 3. パッケージリストの更新
AmazonLinuxのsudo yum update -yに相当

```sh
sudo apt update
```

<aside>

#### 💡 ubuntuとAws Linuxはコマンドが異なる

`yum` は Red Hat系のディストリビューション（CentOS、RHEL、Amazon Linux等）で使われるパッケージマネージャーで、UbuntuはDebian系のディストリビューション。異なるパッケージマネージャーを使用している

##### Ubuntuでの正しいコマンド

###### 1. パッケージリストの更新
```bash
sudo apt update
```

###### 2. システムのアップグレード
```bash
sudo apt upgrade -y
```

###### 3. 一括実行（推奨）
```bash
sudo apt update && sudo apt upgrade -y
```

##### よく使われるaptコマンド

| yumコマンド | aptコマンド | 説明 |
|------------|------------|------|
| `sudo yum update -y` | `sudo apt update && sudo apt upgrade -y` | システム更新 |
| `sudo yum install package` | `sudo apt install package` | パッケージインストール |
| `sudo yum remove package` | `sudo apt remove package` | パッケージ削除 |
| `yum search keyword` | `apt search keyword` | パッケージ検索 |
| `yum list installed` | `apt list --installed` | インストール済みパッケージ一覧 |

##### 使用中のディストリビューションの確認方法
```bash
cat /etc/os-release
```
または
```bash
lsb_release -a
```

これでUbuntuかどうか、バージョンも含めて確認できます。

Ubuntuでシステムを最新に保つには、定期的に `sudo apt update && sudo apt upgrade -y` を実行することをお勧めします。

</aside>

#### 4. サーバーへGitのインストール

```
sudo apt install git -y
```

#### 5. Node.js と npm のインストール

##### 方法A. NodeSourceリポジトリから（推奨・最新LTS版）
例： Node.js 20.x (LTS) をインストール

```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

##### 方法B. nvmを使用（複数バージョン管理が可能）

```
# nvmのインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# シェルを再読み込み
source ~/.bashrc

# 最新LTS版のNode.jsをインストール
nvm install --lts
nvm use --lts
```

#### 6. インストールの確認
```
node --version
npm --version
git --version
```

#### 7. npmのグローバルパッケージ用ディレクトリを設定（権限エラー回避）
```
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

#### 8. ローカルからEC2へプロジェクトを転送
##### 方法A: scpコマンドで直接転送（最短）
ローカルのターミナルで、プロジェクトディレクトリの1つ上の階層で実行：
```sh
scp -i <秘密鍵ファイル> -r lapsy-front ec2-user@<EC2のパブリックIP>:~/
```
##### 方法B: GitHub等のリポジトリを使う場合
1. プロジェクトをGitHub等にpush
2. PushしたリポジトリをEC2でclone
   ```sh
   git clone <リポジトリURL>
   ```

#### 9. EC2で依存パッケージをインストール
```sh
cd ~/lapsy-front
npm install
```

#### 10. 必要に応じて環境変数や設定ファイルを配置

#### 11. サーバー起動（例: Next.js開発サーバー）
```sh
npm run dev
```
または本番用
```sh
npm run build
npm start
```
<!-- 
EC2のUbuntuでNode.js、npm、gitをインストールする方法をご説明します。

## 1. 現在のインストール状況を確認

まず、既にインストールされているかどうかを確認しましょう：

```bash
# Node.jsの確認
node --version

# npmの確認
npm --version

# gitの確認
git --version
```

## 2. パッケージリストを更新

```bash
sudo apt update
```

## 3. Gitのインストール

```bash
sudo apt install git -y
```

## 4. Node.js と npm のインストール

### 方法1: Ubuntu標準リポジトリから（簡単だが古いバージョンの場合が多い）

```bash
sudo apt install nodejs npm -y
```

### 方法2: NodeSourceリポジトリから（推奨・最新LTS版）

```bash
# Node.js 20.x (LTS) をインストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

### 方法3: nvmを使用（複数バージョン管理が可能）

```bash
# nvmのインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# シェルを再読み込み
source ~/.bashrc

# 最新LTS版のNode.jsをインストール
nvm install --lts
nvm use --lts
```

## 5. インストール確認

```bash
node --version
npm --version
git --version
```

## 推奨設定

### npmのグローバルパッケージ用ディレクトリを設定（権限エラー回避）

```bash
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### gitの初期設定

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

## 一括インストールスクリプト例

```bash
#!/bin/bash
# パッケージ更新
sudo apt update

# git インストール
sudo apt install git -y

# Node.js (NodeSource) インストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y

# インストール確認
echo "=== インストール確認 ==="
git --version
node --version
npm --version
```

## 注意点

- **方法2（NodeSource）** が最も推奨されます。最新のLTS版が取得でき、npmも同時にインストールされます
- **方法3（nvm）** は複数のNode.jsバージョンを管理したい場合に便利です
- EC2のセキュリティを考慮し、必要なポートのみを開放してください

これで開発環境の基本的なセットアップが完了します！ -->

### 所要時間の目安

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

---

# 3. GitHub Actionsなどでの自動化・JWTによるセキュア化の設定

## Express APIとの連携 (JWT付き)

<aside>

### 💡 なぜこのプロセスが必要なのか？

主には、 **フロントエンド（Next.js + Supabase）から、EC2上のExpress APIにセキュアにアクセスする** ため

- **Supabaseでログインしていても、Expressサーバーにはセッション情報は共有されない** = 何も対策しなければ誰でもExpressのAPIを叩けてしまう

- Supabaseが発行するJWT（アクセストークン）をヘッダーに付けてリクエスト → Express側でそのJWTを検証し、「これは本当にログイン済みユーザーだ」と判定する

= 「ログイン済ユーザーだけが安全にAPIを使える状態」
を構築するための **本番運用では必須レベルの認証接続手順**

</aside>




