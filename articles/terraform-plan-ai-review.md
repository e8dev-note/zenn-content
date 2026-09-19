---
title: "terraform plan は何を比べているのか【AI の Terraform 暴走を止める #2】"
emoji: "🛑"
type: "tech"
topics: ["terraform", "iac", "ai", "devops"]
published: true
---

AI が書いた Terraform の実行を承認する立場の人に向けて、基本コマンドを 1 本ずつ「機構 / 設計意図 / AI を止めるサイン」の 3 問で解説するシリーズの 2 本目です。1 本目は [terraform init は何をしているのか](https://zenn.dev/e8dev/articles/terraform-init-ai-review)。

今回は `terraform plan`。「実行前の確認」と言われますが、何と何を比べていますか?

@[youtube](1Kinl8WglrA)

ログはすべて `hashicorp/random` provider で取りました。クラウドの課金なしで手元で再現できます(Terraform 1.16.0)。

## 機構: 3 つを突き合わせる

![3 つの突き合わせ](/images/terraform-plan/three-way.png)

plan が比べているのは 3 つです。

- **HCL**(`main.tf`)… あるべき姿
- **state**(`terraform.tfstate`)… Terraform が覚えている台帳
- **現実** … クラウドの実体

順番としては、まず現実を読みに行って state を最新にし(refresh)、その state と HCL の差分を出します。plan の冒頭に出るこの行が、現実を読んでいる証拠です。

```text
$ terraform plan
terraform_data.config: Refreshing state... [id=4443e7b5-57aa-b9f2-5739-2d30db4ce34c]
random_pet.app: Refreshing state... [id=rich-minnow]
random_string.token: Refreshing state... [id=cG!W$Xz7]
```

手作業でクラウド側が変えられていた場合(drift)も、ここで見つかります。

出力は「これから何をするか」の一覧です。最後の `Plan:` 行に追加・変更・削除の数が出ます。

```text
Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create

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
```

差分がなければこうなります。

```text
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.
```

plan 自体は何も作らず、何も消しません。読むだけのコマンドです。

### 記号は 4 つ

冒頭の凡例に、その plan で使われる記号が出ます。

```text
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
  - destroy
-/+ destroy and then create replacement
```

`+` 作成、`~` その場で更新、`-` 削除、`-/+` 削除してから作り直し。`-/+` は実体が一度消えるので、読み方は次の記事で 1 本使って扱います。

## 設計意図: なぜ plan と apply が分かれているのか

![なぜ分かれているか](/images/terraform-plan/why-separate.png)

**読む人と実行する人を分けるため**です。plan は読むだけなので、出力を Pull Request に貼れば、レビュアーは「何が起きるか」を実行前に読めます。承認された plan だけが apply に進む。この分業が Terraform の安全装置の中心にあります。

ただし、plan の末尾には毎回こう書いてあります。

```text
Note: You didn't use the -out option to save this plan, so Terraform can't
guarantee to take exactly these actions if you run "terraform apply" now.
```

plan と apply の間に HCL や現実が変われば、apply は別の内容を実行します。「読んだものがそのまま実行される」を保証したいときは `-out` で plan をファイルに保存し、apply にそのファイルを渡します。

```text
$ terraform plan -out=tfplan
Saved the plan to: tfplan

To perform exactly these actions, run the following command to apply:
    terraform apply "tfplan"
```

## AI を止めるサイン

![サイン 1](/images/terraform-plan/sign-1.png)

| # | AI がやろうとしていること | 何が起きるか | 聞くこと |
|---|---|---|---|
| 1 | plan の出力を読まずに「問題ありません」と要約して apply へ進む | replace(作り直し)や destroy が要約の数字に隠れる | 「`Plan:` の行の数字と、replace はあった?」 |
| 2 | plan を見せた後に HCL を書き換えてから apply する | 実行される内容が、レビューした plan と別物になる | 「その plan は今の HCL のもの? `-out` で保存した?」 |
| 3 | CI の plan ジョブと apply ジョブで plan ファイルを受け渡していない | apply は plan をやり直すので、レビューした内容と実行内容がズレうる | 「apply は保存した plan ファイルを使っている?」 |

### サイン 1 について

AI は plan の出力を要約してくれます。それ自体は便利ですが、`Plan: 2 to add, 1 to change, 2 to destroy.` を「リソースを更新します」と言い換えることがある。destroy に数えられた 2 つが何かは、要約からは分かりません。plan の本文で `# ... will be destroyed` と `# ... must be replaced` の行を探してください。

### サイン 2 について

plan を見せてから「ついでにこれも直しておきます」と HCL を変え、そのまま apply に進む動きです。レビューした plan は古い HCL のもの。apply は新しい HCL で plan をやり直すので、承認していない変更が実行されます。plan を出したら HCL を触らせない、あるいは `-out` で固定して、apply にはそのファイルを渡す。

### サイン 3 について

サイン 2 の CI 版です。PR で plan を出して人が読み、merge 後に apply する構成はよくありますが、apply ジョブが plan ファイルを受け取らず plan をやり直していると、読んだ内容と実行内容が一致する保証がありません。plan ジョブの成果物(artifact)として plan ファイルを渡すのが標準形です。

## まとめ

- plan は **HCL・state・現実** の 3 つを突き合わせ、「これからやること」を見せるだけ。実体には触らない
- plan の出力は人が読む証拠。PR に貼る
- 読んでから apply。固定したいなら `-out`

## シリーズ

1. [terraform init は何をしているのか](https://zenn.dev/e8dev/articles/terraform-init-ai-review)
2. terraform plan は何を比べているのか(この記事)
3. plan の記号を読む: `-/+` は一度消える
4. terraform apply で何が起きるのか
5. state とは何か: Terraform の「台帳」
6. remote state と lock: 台帳を共有する

次は plan の記号。`-/+` を見逃すと、本番のデータベースが一度消えます。
