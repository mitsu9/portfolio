---
title:      "GCP Cloud Functionsで秘匿情報を扱う"
date:       2020-01-25T15:00:00+09:00
categories: ["tech"]
tags:       ["GCP"]
---

何か外部とやりとりをするコードを書くといつも付いてくるのが秘匿情報の管理だと思います。

最近GCPを勉強しているのですが、今回利用した秘匿情報を扱う方法について紹介したいと思います。

## 今回作ったもの

Spotify APIから再生している曲の情報を取得しSlackのStatusに設定するという処理をCloud Functionsで実装しました。
ここでは具体的な実装については触れませんが、気になる方は以下のリポジトリを見てください。

[slack-spotify-status](https://github.com/mitsu9/slack-spotify-status)

さて、これを実装するために必要な秘匿情報としては以下のものがあります。

- SpotifyAPI: AccessToken, RefreshToken
- SlackAPI: AccessToken

SpotifyはOAuth2を使用してAPIを利用しています。
AccessTokenには期限があり、期限切れの場合RefreshTokenを用いて再度AccessTokenを取得するのですが、このRefreshTokenも定期的に更新されます。
そのため、このTokenの更新を保存できる仕組みを利用する必要がありました。

秘匿情報を環境変数に設定する方法はよく使われると思いますが、今回は秘匿情報の更新をおこなう必要があったので別の方法を利用しました。

## Cloud Storage + Cloud KMSを用いて秘匿情報を扱う

今回はCloud StrageとCloud KMSを用いて秘匿情報を管理するようにしました。

まずはあらかじめCloud KMSに保管された鍵で秘匿情報を暗号化しCloud Storageに置いておきます。

Cloud Functionsでは実行時にCloud Storageから秘匿情報ファイルを取得し、Cloud KMSを用いて複号します。
そして、この復号された秘匿情報ファイルを読み込み利用します。
また、秘匿情報に更新があった際にはCloud KMSを用いて暗号化しCloud Storageにアップロードします。

この秘匿情報を利用する流れをまとめた図が以下のものになります。

{{< figure src="/img/gcp_secret/1.jpg" class="center" width="400" >}}

Cloud FunctionsではCloud Storageからファイルを取得しCloud KMSで復号をおこないますが、この処理は使用する言語によって変わってくるのでここでは割愛します。
以下では、あらかじめCloud Storageに暗号化されたファイルを保管する方法と、Cloud Functionsのデプロイについて説明します。

## 秘匿情報を暗号化してCloud Storageに保管する

Cloud KMSを用いた暗号化とCloud Storageへの保存は以下のコマンドで実行することができます。

ローカル等で必要な秘匿情報ファイルを作成し、以下のコマンドを実行します。


```sh
# 鍵の作成
$ gcloud kms keyrings create KEY_RING_NAME --location LOCATION
$ gcloud kms keys create KEY_NAME \
  --location LOCATION \
  --keyring KEY_RING_NAME \
  --purpose encryption
# 暗号化
$ gcloud kms encrypt \
  --location LOCATION \
  --keyring KEY_RING_NAME \
  --key KEY_NAME \
  --plaintext-file config.toml \
  --ciphertext-file config.toml.enc
# バケットへ保存
$ gsutil mb gs://BUCKET/
$ gsutil cp config.toml.enc gs://BUCKET/
```

## Cloud Functionsのデプロイ
Cloud Functionsをデプロイする際には環境変数で以下の情報を与える必要があります。

- Cloud Strage
  - bucket
  - file name
- Cloud KMS
  - project id
  - location
  - key ring name
  - key name

Cloud Functionsをデプロイする際に環境変数を設定する方法は2種類あり、コマンドの中で指定する方法とファイルを指定する方法があります。
今回は設定する環境変数が多いため、ファイルを指定する方法を使用しました。

```yaml
# env.yaml
BUCKET: bucket-name
FILE_NAME: file-name
PROJECT_ID: project-id
LOCATION: location
KEY_RING_NAME: key-ring-name
KEY_NAME: key-name
```

```sh
$ gcloud functions deploy FUNCTION_NAME --env-vars-file env.yaml <other options>
```

## 参考リンク
- https://cloud.google.com/kms/docs/store-secrets?hl=ja
- https://cloud.google.com/functions/docs/env-var?hl=ja
- https://cloud.google.com/kms/docs/creating-keys?hl=ja#kms-create-keyring-cli
- https://cloud.google.com/sdk/gcloud/reference/kms/encrypt
