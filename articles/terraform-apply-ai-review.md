---
title: "terraform apply で何が起きるのか【AI の Terraform 暴走を止める #4】"
emoji: "🛑"
type: "tech"
topics: ["terraform", "iac", "ai", "devops"]
published: false
---

AI が書いた Terraform の実行を承認する立場の人に向けて、基本コマンドを 1 本ずつ「機構 / 設計意図 / AI を止めるサイン」の 3 問で解説するシリーズの 4 本目です。前回は [plan の記号を読む: -/+ は一度消える](https://zenn.dev/e8dev/articles/terraform-plan-symbols-ai-review)。

今回は `terraform apply`。`yes` と打ったあと、何が起きているか説明できますか?

@[youtube](VIDEO_ID_APPLY)

ログはすべて `hashicorp/random` provider で取りました。クラウドの課金なしで手元で再現できます(Terraform 1.16.0)。

## 機構: 承認 → 実行 → state 更新

### まず plan を作り直し、承認を求める

apply は最初に plan をやり直します。前回 plan を見ていても、apply は自分で plan を作り直して、それを見せてから承認を求めます。

```text
$ terraform apply

Terraform will perform the following actions:

  # random_pet.app will be created
  + resource "random_pet" "app" {
      + id        = (known after apply)
      + length    = 2
      + separator = "-"
    }

  # terraform_data.config will be created
  + resource "terraform_data" "config" {
      + id     = (known after apply)
      + input  = "v1"
      + output = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

`Only 'yes' will be accepted`。`y` も `Yes` も拒否されます。ここが、人が plan を読む唯一の関所です。

### 承認後、依存関係の順に実行する

```text
terraform_data.config: Creating...
terraform_data.config: Creation complete after 0s [id=4443e7b5-57aa-b9f2-5739-2d30db4ce34c]
random_pet.app: Creating...
random_pet.app: Creation complete after 0s [id=rich-minnow]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

HCL に書いた順ではなく、依存関係の順です。依存し合わないリソースは並列に走ります。**1 つ終わるごとに、その結果を state に書き込みます**。

### 終わると terraform.tfstate が更新される

![apply 後](/images/terraform-apply/tfstate.png)

```json:terraform.tfstate
{
  "version": 4,
  "terraform_version": "1.16.0",
  "serial": 3,
  "lineage": "2ba50235-d5e7-b713-664a-69edf99ff6c5",
  "resources": [
    {
      "mode": "managed",
      "type": "random_pet",
      "name": "app",
      "instances": [
        { "attributes": { "id": "rich-minnow", "length": 2, "separator": "-" } }
      ]
    }
  ]
}
```

これが「Terraform が管理している」の実体です。state の中身は次の記事で 1 本使って扱います。

## 設計意図: なぜ 1 つずつ state に書くのか

![途中で失敗したら](/images/terraform-apply/partial-failure.png)

途中で失敗しても「どこまで作ったか」を失わないためです。リソース 1 が成功してリソース 2 で失敗したら、state には 1 だけ記録されている。原因を直して次の apply をすれば、1 は「変更なし」、2 から続きになります。

裏を返すと、**失敗した直後の state は「途中の状態」**です。半分できている。この事実を知らずに動くと事故になります。

### -out で保存した plan を渡す

前回の記事で触れた `-out` を apply に渡すと、承認のプロンプトは出ません。代わりに「読んだものがそのまま実行される」が保証されます。

```text
$ terraform plan -out=tfplan

  # terraform_data.config will be updated in-place
  ~ resource "terraform_data" "config" {
        id     = "4443e7b5-57aa-b9f2-5739-2d30db4ce34c"
      ~ input  = "v2" -> "v3"
      ~ output = "v2" -> (known after apply)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

Saved the plan to: tfplan

To perform exactly these actions, run the following command to apply:
    terraform apply "tfplan"

$ terraform apply tfplan
terraform_data.config: Modifying... [id=4443e7b5-57aa-b9f2-5739-2d30db4ce34c]
terraform_data.config: Modifications complete after 0s [id=4443e7b5-57aa-b9f2-5739-2d30db4ce34c]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

承認を飛ばしてよいのは、plan ファイルを人が読んで承認した後だけ、という前提です。CI で「PR で plan → merge 後に apply」を組むときの標準形がこれです。

### destroy は apply の裏返し

```text
$ terraform plan -destroy

  # random_integer.port will be destroyed
  # random_pet.app will be destroyed
  # terraform_data.config will be destroyed

Plan: 0 to add, 0 to change, 3 to destroy.
```

`terraform destroy` は、state にあるものを全部消す plan を作って、同じ承認フローで実行します。「クリーンアップします」と言って destroy に進もうとしたら、先に `plan -destroy` の出力を見てください。

## AI を止めるサイン

![サイン 1](/images/terraform-apply/sign-1.png)

| # | AI がやろうとしていること | 何が起きるか | 聞くこと |
|---|---|---|---|
| 1 | `terraform apply -auto-approve` を実行する | 承認のステップ = 人が plan を読む唯一の関所を飛ばしている | 「その apply、誰が plan を読んだ?」 |
| 2 | エラーで止まった apply を、原因を見ずに再実行する | state は途中の状態。原因によっては二重作成や意図しない削除が走る | 「エラーの原因は? `state list` に何が残っている?」 |
| 3 | apply の途中で Ctrl+C を連打して止める | 1 回目は安全な中断(進行中の処理を待つ)。2 回目は強制終了で state が壊れうる | 「止めるなら 1 回。終わるまで待てる?」 |

### サイン 1 について

`-auto-approve` は「承認を求めない」フラグです。AI エージェントは対話プロンプトで止まるのを嫌うので、自分の判断でこれを付けることがあります。付けた瞬間、plan を読むステップが消えます。使ってよいのは、人が承認した plan ファイルを CI の apply ジョブが実行するときだけです。

### サイン 2 について

apply がエラーで止まると、AI は「もう一度実行します」と言いがちです。ただし、その時点で state は途中の状態。エラーの原因が一時的なもの(API のタイムアウトなど)なら再実行で続きから進みますが、原因が「作成は成功したのに state への書き込みで失敗した」だった場合、再実行は同じリソースをもう 1 つ作ります。原因を読んで、`terraform state list` で何が記録されているかを見てから再実行してください。

### サイン 3 について

Terraform は 1 回目の Ctrl+C を受け取ると、新しい操作を始めるのをやめ、進行中の操作が終わるのを待ってから state を保存して終了します。2 回目の Ctrl+C は強制終了で、進行中の操作の結果が state に書かれないまま終わります。実体はできたのに state にない、というリソースが生まれます。止めるなら 1 回押して待つ。

## まとめ

- apply は plan を作り直して承認を取り、依存順に実行し、1 つずつ state に記録する
- 失敗しても state は「途中まで」を覚えている。再実行の前に原因と `state list` を見る
- `-auto-approve` は CI の apply ジョブ以外で使わない

## シリーズ

1. [terraform init は何をしているのか](https://zenn.dev/e8dev/articles/terraform-init-ai-review)
2. [terraform plan は何を比べているのか](https://zenn.dev/e8dev/articles/terraform-plan-ai-review)
3. [plan の記号を読む: `-/+` は一度消える](https://zenn.dev/e8dev/articles/terraform-plan-symbols-ai-review)
4. terraform apply で何が起きるのか(この記事)
5. state とは何か: Terraform の「台帳」
6. remote state と lock: 台帳を共有する

次は state。AI が「壊れているから消してやり直す」と言ったら、止めてください。
