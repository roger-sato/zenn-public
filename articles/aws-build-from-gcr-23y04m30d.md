---
title: "AWSのCodeBuildでGCRにあるイメージを使う"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "GCR", "Docker"]
published: true
---

CodeBuild で下のような GCR にあるイメージをベースとする Dockerfile のビルドをしたかったんですが
それっぽい記事がなかったので書いてみました(ちゃんと調べてないだけかもですが)

```dockerfile
FROM gcr.io/project/image

...

```

## 忙しい人向け

以下のようなケースのやり方を紹介します

- AWS の CodeBuild でビルドする Dockerfile のベースイメージが GCR にある
- GCR のベースイメージのリポジトリが private になっている

こんな感じにやります

- GCP にサービスアカウントを作成する
- サービスアカウントのアクセスキーをパラメタストアに保存する
- buildspec.yml 内でパラメタストアから認証情報を取り出す
- 取り出した認証情報を使って docker login を行う
- そのあとは通常通りビルドを行う

## GCP の認証情報を用意する

まず GCP のコンソールに入って、IAM と管理 > サービスアカウントからサービスアカウントを作成します
サービスアカウントには Storage オブジェクト閲覧者(roles/storage.objectViewer)のロールを付与します

![](/images/aws-build-from-gcr/service-account.png)

作成されたサービスアカウントを選択して キー タブを開き、json タイプを指定して新しい鍵を作成を行います
鍵を作成すると json ファイルががダウンロードされます。後ほど、このファイルを使います

![](/images/aws-build-from-gcr/generate-key.png)

## 認証情報をパラメータストアに入れる

AWS の Parameter Store を新しく作成して、認証情報を格納していきます
Type を SecureString にして、Value のところに先ほど取得した JSON ファイルの中身を全て貼り付けます
最後に作成ボタンをクリックすれば完了です

![](/images/aws-build-from-gcr/store-parameter-store.png)

## CodeBuild で認証を通す

まずは、env セクションに parameter-store から認証情報を取得します
以下は、buildspec.yml の例です

```yaml
env:
  parameter-store:
    DOCKER_USER: "DOCKER_USER_NAME"
    DOCKER_PASSWORD: "DOCKER_PASSWORD"
    GCR_CREDENTIAL: "GCR_CREDENTIAL"
```

次に認証情報を使って GCR にログインしていきます
GCR だけでなく、AWS の ECR に並行してログインすることができます

```yaml
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo Logging in to GoogleContainer Registry...
      - echo $GCR_CREDENTIAL | docker login -u _json_key --password-stdin https://gcr.io
```

最後に通常通りビルドを行えば大丈夫です

```yaml
build:
  commands:
    - docker build -f app/Dockerfile .
```

全体のコードはこんな感じです

```yaml
version: 0.2

env:
  parameter-store:
    DOCKER_USER: "DOCKER_USER_NAME"
    DOCKER_PASSWORD: "DOCKER_PASSWORD"
    GCR_CREDENTIAL: "GCR_CREDENTIAL"

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo Logging in to Google Container Registry...
      - echo $GCR_CREDENTIAL | docker login -u _json_key --password-stdin https://gcr.io
  build:
    commands:
      - docker build -f app/Dockerfile .
```
