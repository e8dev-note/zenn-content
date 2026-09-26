---
title: "terraform init は何をしているのか【AI の Terraform 暴走を止める #1】"
emoji: "🛑"
type: "tech"
topics: ["terraform", "iac", "ai", "devops"]
published: true
---

AI エージェントに Terraform を触らせるようになって、レビューの中身が変わりました。書かれた HCL を読むより先に、**AI が打とうとしているコマンドと、その出力の読み方**を見る必要が出てきた。エラーを消すために lock file を削除する、plan を読まずに apply に進む、lock エラーに `force-unlock` で応える。どれも人間の初心者と同じ間違いですが、AI は速いので、気づいたときには終わっています。

このシリーズでは、Terraform の基本コマンドを 1 本ずつ、「機構 / 設計意図 / AI を止めるサイン」の 3 問で解説します。対象は「自分では Terraform をあまり書かないが、AI が書いた Terraform の実行を承認する立場にいる人」。エンジニア・PdM・レビュアー。

1 本目は `terraform init`。AI が最初に打つコマンドです。何をしているか、説明できますか?

@[youtube](4f7iiYARaJk)

ログはすべて `hashicorp/random` provider で取りました。クラウドの課金なしで手元で再現できます(Terraform 1.16.0)。

## 機構: init がやること 3 つ

```text
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/random versions matching "~> 3.6"...
- Installing hashicorp/random v3.9.1...
- Installed hashicorp/random v3.9.1 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
```

出力の順に 3 つです。

### ① backend の初期化

`Initializing the backend...` の行。state(Terraform が管理しているものの台帳)の置き場所を決めます。何も書かなければローカルの `terraform.tfstate`。S3 などに置く設定があればそこに繋ぎます。

### ② provider の取得

`Initializing provider plugins...` 以下。`required_providers` の制約(ここでは `~> 3.6`)の範囲でレジストリからバージョンを選び、バイナリをダウンロードして `.terraform/` に置きます。

### ③ lock file の生成

![lock file](/images/terraform-init/lockfile.png)

選んだバージョンとハッシュを `.terraform.lock.hcl` に記録します。

```hcl:.terraform.lock.hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/random" {
  version     = "3.9.1"
  constraints = "~> 3.6"
  hashes = [
    "h1:PlW+UZ4EElQF3NQwf41KQwavFujab3Czc51zu9dyVM8=",
    "zh:05f4734c1f0be840b711b3eff259ebc5fca436784c728955b1678078466f48d7",
    ...
  ]
}
```

`constraints` が HCL に書いた条件、`version` が実際に選ばれた版、`hashes` がそのバイナリの指紋です。

## 設計意図: なぜ lock file があるのか

![なぜ lock file か](/images/terraform-init/why-lockfile.png)

CI や同僚の環境でも同じ provider が入るためです。`~> 3.6` という制約だけでは、明日 3.10.0 が出たら別の版が入る。lock file を commit しておけば、誰がいつ init しても 3.9.1 になります。`package-lock.json` と同じ発想です。

2 回目の init はこのファイルを読んで、同じ版を再利用します。

```text
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/random from the dependency lock file
- Using previously-installed hashicorp/random v3.9.1

Terraform has been successfully initialized!
```

出力に `Reusing previous version` とあれば正常です。

## AI を止めるサイン

![サイン 1](/images/terraform-init/sign-1.png)

| # | AI がやろうとしていること | 何が起きるか | 聞くこと |
|---|---|---|---|
| 1 | エラーを消すために `.terraform.lock.hcl` を削除して init し直す | provider のバージョンが黙って変わる | 「なぜ lock file を消す必要があった?」 |
| 2 | `terraform init -upgrade` を何気なく実行する | 制約の範囲で最新版に更新される | 「今 upgrade が必要な理由は?」 |
| 3 | `.terraform/` を `git add` する | 中身は OS/CPU 別の provider バイナリ(17MB)。環境ごとに違う | 「`.gitignore` に `.terraform/` は入っている?」 |
| 4 | backend 変更のエラーに `-reconfigure` で応じる | `-reconfigure` は state を引き継がない。`-migrate-state` なら移行する | 「既存の state はどこに行く?」 |

### サイン 1 が出る場面

よくあるのがこのエラーです。

```text
$ terraform plan

Error: Required plugins are not installed

The installed provider plugins are not consistent with the packages selected
in the dependency lock file:
  - registry.terraform.io/hashicorp/random: there is no package for
    registry.terraform.io/hashicorp/random 3.9.1 cached in .terraform/providers
```

clone 直後や `.terraform/` を消した後に出ます。`.terraform/` が無いと plan は動かない。ここで必要なのは `terraform init` であって、lock file の削除ではありません。エラー文にも `dependency lock file` とあるので、AI はここに反応して lock file を消し始めることがあります。消して init し直せばエラーは消えますが、provider の版が変わっています。

### サイン 2 について

`-upgrade` そのものは悪くありません。provider を上げたいときに使うコマンドです。問題は「何気なく」の部分。lock file の版が変わるので、その diff が PR に含まれます。「今回の変更に provider の更新は含まれるのか」を確認してください。

### サイン 4 について

backend の設定を変えると、次の init で「backend が変わった」というエラーが出ます。選択肢は 2 つで、`-migrate-state` は今の state を新しい置き場所に移し、`-reconfigure` は移さず、新しい置き場所にある state(なければ空)で始めます。元の state は元の場所に残ったままです。AI が `-reconfigure` を選んだら、既存の state がどこに行くかを聞いてください。空の state で plan すると、全リソースが「新規作成」になります。

## まとめ

- init の実体は **backend・provider・lock file** の 3 つ
- `.terraform.lock.hcl` は commit する。`.terraform/` は commit しない
- `Required plugins are not installed` に必要なのは init。lock file の削除ではない

## シリーズ

1. terraform init は何をしているのか(この記事)
2. [terraform plan は何を比べているのか](https://zenn.dev/e8dev/articles/terraform-plan-ai-review)
3. [plan の記号を読む: `-/+` は一度消える](https://zenn.dev/e8dev/articles/terraform-plan-symbols-ai-review)
4. [terraform apply で何が起きるのか](https://zenn.dev/e8dev/articles/terraform-apply-ai-review)
5. state とは何か: Terraform の「台帳」
6. remote state と lock: 台帳を共有する

動画は 1 本 2 分弱で、台本は「機構 / 設計意図 / AI を止めるサイン」の 3 問で固定しています。次は `terraform plan`。「実行前の確認」と言うけれど、何と何を比べているのか。
