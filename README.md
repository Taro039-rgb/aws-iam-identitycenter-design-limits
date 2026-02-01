# AWS IAM Identity Center  
## Permission Set 共有時の AssumeRole 設計制約 検証ポートフォリオ

---

## 概要

本ポートフォリオは、AWS IAM Identity Center（旧 AWS SSO）において  
**同一 Permission Set を複数ユーザーに割り当てた場合の AssumeRole 制御の限界**を  
実機検証と CloudTrail の監査ログから明確化した検証記録である。

結論として、AssumeRole の制御単位は **ユーザーではなく Permission Set** であり、  
Trust Policy ではユーザー単位の制御ができないことを確認した。

本検証は、実運用で発生しやすい  
「同一 Permission Set を共有する複数の SSO ユーザーに対して、  
ユーザー単位で AssumeRole 先を制限したい」という要求について、  
なぜそれが設計上不可能なのか、またどのように回避すべきかを  
説明可能な状態にすることを目的としている。


---

## 検証構成

### AWS Organizations 構成

- AWS Organizations 配下
- Sandbox OU に検証用アカウントを配置
- SCP は FullAWSAccess のみ適用（検証影響を最小化）

### IAM Identity Center 構成

- Permission Set：ps-switch-verify
- ユーザー：
  - sso-test-user-a
  - sso-test-user-b
- 両ユーザーに **同一 Permission Set** を割り当て

---

## Permission Set 設計

### インラインポリシー（最小構成）

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": "sts:AssumeRole",
          "Resource": "*"
        }
      ]
    }

本 Permission Set は「入口用」として設計し、  
作業権限は AssumeRole 先の IAM Role に集約する。

---

## AWSReservedSSO ロールの生成

Permission Set をアカウントに割り当てると、  
以下の IAM Role が AWS により自動生成される。

    AWSReservedSSO_ps-switch-verify_<hash>

このロールが AssumeRole の **Principal（入口ロール）**となる。

重要な点として、  
**同一 Permission Set を割り当てられたユーザーは、  
すべてこのロールを経由して AWS にアクセスする。**

---

## スイッチ先ロール作成

### ロール名

    role-work-x

### 許可ポリシー

    PowerUser

### Trust Policy

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::<account-id>:role/aws-reserved/sso.amazonaws.com/ap-northeast-1/AWSReservedSSO_ps-switch-verify_<hash>"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }

---

## 検証①：AssumeRole 動作確認

- sso-test-user-a でログインし、role-work-x にスイッチ可能
- sso-test-user-b でログインしても、同様に role-work-x にスイッチ可能

ユーザーが異なっていても、  
**同一 Permission Set 経由であれば同一ロールに AssumeRole できる**ことを確認。

---

## 検証②：CloudTrail による証跡確認

### Event name

    AssumeRole

### userIdentity（user-a）

    {
      "type": "AssumedRole",
      "arn": "arn:aws:sts::<account-id>:assumed-role/AWSReservedSSO_ps-switch-verify_<hash>/sso-test-user-a"
    }

### userIdentity（user-b）

    {
      "type": "AssumedRole",
      "arn": "arn:aws:sts::<account-id>:assumed-role/AWSReservedSSO_ps-switch-verify_<hash>/sso-test-user-b"
    }

### sessionIssuer（共通）

    {
      "type": "Role",
      "arn": "arn:aws:iam::<account-id>:role/aws-reserved/sso.amazonaws.com/ap-northeast-1/AWSReservedSSO_ps-switch-verify_<hash>"
    }

CloudTrail 上では、  
**ユーザー名は異なるが、Principal は同一 AWSReservedSSO ロール**として記録される。

---

## エビデンス

本検証で使用した証跡は、以下のディレクトリに整理して格納している。

### CloudTrail（AssumeRole イベント）

- user-a による AssumeRole  
  `/evidence/01-cloudtrail-assumerole-user-a.png`

- user-b による AssumeRole  
  `/evidence/02-cloudtrail-assumerole-user-b.png`

いずれのイベントにおいても、  
sessionIssuer が同一の AWSReservedSSO ロールであることを確認。

### SSO ログイン後のコンソール

- user-a ログイン後  
  `/evidence/03-sso-console-user-a.png`

- user-b ログイン後  
  `/evidence/04-sso-console-user-b.png`

### Trust Policy / 入口ロール確認

- Switch 先ロールの Trust Policy  
  `/evidence/05-switchrole-trust-policy.png`

- AWSReservedSSO ロール ARN  
  `/evidence/06-awsreserved-sso-role-arn.png`

---

## 結論（設計上の制約）

- 同一 Permission Set を割り当てたユーザーは  
  AWS 側では **同一の AWSReservedSSO ロールとして扱われる**
- Trust Policy では Principal として AWSReservedSSO ロールのみが評価されるため、  
同一 Permission Set を共有する複数の SSO ユーザーを  
ユーザー単位で識別して AssumeRole 制御することはできない
- AssumeRole の制御単位は **Permission Set**

---

## 回避策

本検証の事象は、同一 Permission Set を共有したことで
AWSReservedSSO ロールが同一となり、
Trust Policy でユーザー単位の識別ができないことが原因である。

### 回避策：Permission Set を分割する

ユーザー／職務ごとに Permission Set を分割し、
入口となる AWSReservedSSO ロールを分離することで回避できる。

※ 実運用では、Permission Set は入口（AssumeRole）のみに限定し、
作業権限はアカウント内の IAM Role に集約する構成が一般的である。



---

## 補足

- CloudTrail のイベント履歴はリージョン単位で記録される
- 本ポートフォリオでは accessKeyId 等の一時認証情報はマスキング済み

---

## まとめ

IAM Identity Center における AssumeRole 制御は、  
**ユーザー単位ではなく Permission Set 単位で設計する必要がある。**

本ポートフォリオは、  
その設計制約を **実機操作＋監査ログ**で説明可能な形に整理したものである。
