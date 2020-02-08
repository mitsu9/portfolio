---
title:      "AWS STSで一時的な認証情報を取得する&効率化のための設定"
date:       2020-02-08T13:00:00+09:00
categories: ["tech"]
tags:       ["AWS", "STS"]
---

AWSのIAM Userを使う際にはMFAを有効にして二段階認証をすることは当たり前と考えられるようになっていると思います。

MFAを有効にしているとAWS CLIを使う際にSTSで一時的な認証情報を取得し、その認証情報を用いて各種リソースにアクセスすることになります。

今回はSTSを使って認証情報を取得する方法と、その取得を楽にするために設定していることを紹介します。

## AWS STSを用いて一時的な認証情報を取得する

一時的な認証情報を取得するためにはsts get-session-tokenコマンドを利用します。

以下のようにMFAデバイスと認証コードを与えて実行すると、AccessKeyId・SecretAccessKey・SessionTokenの3つを得ることができます。

```
$ aws sts get-session-token --serial-number arn-of-the-mfa-device --token-code code-from-token
{
    "Credentials": {
        "SecretAccessKey": "secret-access-key",
        "SessionToken": "temporary-session-token",
        "Expiration": "expiration-date-time",
        "AccessKeyId": "access-key-id"
    }
}
```

この3つの情報を環境変数に設定することでAWS CLIを使ってアクセスできるようになります。

```
$ export AWS_ACCESS_KEY_ID=example-access-key-as-in-previous-output
$ export AWS_SECRET_ACCESS_KEY=example-secret-access-key-as-in-previous-output
$ export AWS_SESSION_TOKEN=example-session-token-as-in-previous-output
```

## 認証情報の取得を楽にするスクリプト

コマンドを実行して環境変数に設定するだけとはいえ、毎日のように実行しているとこの手間が面倒に感じてきます。

そこでこの取得から設定までを楽にするための関数をzshで書きました。
実行するとMFAデバイスとコードを聞かれるので答えると認証情報を環境変数に設定してくれます。

MFAデバイスはハードコードしても良いのですが、複数のAWSアカウントを利用するため入力するようにしています。

```
# .zshrc
function set-aws-sts-session-token {
  unset AWS_ACCESS_KEY_ID
  unset AWS_SECRET_ACCESS_KEY
  unset AWS_SESSION_TOKEN

  read SERIAL_NUMBER\?'Input MFA Serial Number: '
  read TOKEN_CODE\?'Input MFA Code: '

  OUTPUT=`aws sts get-session-token \
    --serial-number ${SERIAL_NUMBER} \
    --token-code ${TOKEN_CODE}`

  AWS_ACCESS_KEY_ID=`echo $OUTPUT | jq -r .Credentials.AccessKeyId`
  AWS_SECRET_ACCESS_KEY=`echo $OUTPUT | jq -r .Credentials.SecretAccessKey`
  AWS_SESSION_TOKEN=`echo $OUTPUT | jq -r .Credentials.SessionToken`

  export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
  export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
  export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN

  echo Set envs
  echo AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
  echo AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
  echo AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN
}
```

複数のAWSアカウントを利用しているためコマンド実行前にはprofileを指定して利用しています。

環境変数をunsetしてからSTSのリクエストを発行しているので、AWSアカウントを切り替えることも簡単です。
(このunsetを忘れるとSTSのリクエストが通らなくなります。）

```
$ export AWS_PROFILE=profile
$ set-aws-sts-session-token
Input MFA Serial Number: MFA_SERIAL_NUMBER
Input MFA Code: MFA_CODE
Set envs
AWS_ACCESS_KEY_ID=XXX
AWS_SECRET_ACCESS_KEY=XXX
AWS_SESSION_TOKEN=XXX

# 別のアカウントに切り替えたい時
$ export AWS_PROFILE=profile2
$ set-aws-sts-session-token
```

## 参考リンク
- https://aws.amazon.com/jp/premiumsupport/knowledge-center/authenticate-mfa-cli/
