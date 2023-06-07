---
title: "【MacOS】AWS SSMでエラーが出る時の対処法"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["awsssm", "ssh"]
published: true
---

## エラー内容

ssm で以下のエラーが出ました。

```sh
$ aws ssm start-session --profile $PROFILE --region $REGION --target $INSTANCE_ID

An error occurred (AccessDeniedException) when calling the TerminateSession operation: User: arn:aws:sts::000000000000:assumed-role/AWSReservedSSO_xxxxxxxxxxx/ooooooo@mmmmm.com is not authorized to perform: ssm:TerminateSession on resource: arn:aws:sts::000000000000:assumed-role/AWSReservedSSO_xxxxxxxxxxx/ooooooo@mmmmm.com-xxxxxxxxxxxxxxxxx because no identity-based policy allows the ssm:TerminateSession action
```

## 対応策

結論としては、`session-manager-plugin`をインストールしたら解消しました。

```sh
$ brew install --cask session-manager-plugin
```

## 調査方法

エラー自体は`ssm:TerminateSession`が許可されないという内容ですが、これが原因ではなく、何かがうまくいかずセッションを切るときのエラーが出力されたものと推測しました。
なので、直接の原因を調べるために`--debug` オプションをつけて再度実行します。

```sh
$ aws ssm start-session --debug --profile $PROFILE --region $REGION --target $INSTANCE_ID
```

ログが大量に流れますが、関係ないものが多いのでエラーの部分を検索します。

```sh
2022-03-28 18:47:08,627 - MainThread - awscli.customizations.sessionmanager - DEBUG - SessionManagerPlugin is not present
Traceback (most recent call last):
  File "awscli/customizations/sessionmanager.py", line 83, in invoke
  File "subprocess.py", line 359, in check_call
  File "subprocess.py", line 340, in call
  File "subprocess.py", line 858, in __init__
  File "subprocess.py", line 1706, in _execute_child
FileNotFoundError: [Errno 2] No such file or directory: 'session-manager-plugin'
```

session-manage-plugin がないことがわかりました
