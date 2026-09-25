---
title: "plan の記号を読む: -/+ は一度消える【AI の Terraform 暴走を止める #3】"
emoji: "🛑"
type: "tech"
topics: ["terraform", "iac", "ai", "devops"]
published: true
---

AI が書いた Terraform の実行を承認する立場の人に向けて、基本コマンドを 1 本ずつ「機構 / 設計意図 / AI を止めるサイン」の 3 問で解説するシリーズの 3 本目です。前回は [terraform plan は何を比べているのか](https://zenn.dev/e8dev/articles/terraform-plan-ai-review)。

今回は plan の出力に出る記号。特に `-/+`。この記号を見逃すと、本番のデータベースが一度消えます。

@[youtube](tdoo9ahCJbI)

ログはすべて `hashicorp/random` provider で取りました。クラウドの課金なしで手元で再現できます(Terraform 1.16.0)。

## 機構: 記号は 4 つ

![4 つの記号](/images/plan-symbols/four-symbols.png)

plan の冒頭に、その plan で使われる記号の凡例が出ます。

```text
$ terraform plan

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
  - destroy
-/+ destroy and then create replacement
```

| 記号 | 意味 | 実体は |
|---|---|---|
| `+` | 作成 | 新しくできる |
| `~` | その場で更新 | 残る |
| `-` | 削除 | 消える |
| `-/+` | 削除してから作り直し | **一度消える** |

### -/+ replace

```text
  # random_pet.app must be replaced
-/+ resource "random_pet" "app" {
      ~ id        = "rich-minnow" -> (known after apply)
      ~ length    = 2 -> 3 # forces replacement
        # (1 unchanged attribute hidden)
    }
```

`must be replaced` が見出し、`# forces replacement` が原因です。どの属性が作り直しを強制したか、plan が教えてくれます。ここでは `length` を 2 から 3 に変えたことが原因で、`id` は `(known after apply)`、つまり作り直した後に新しい値になります。

### ~ update in-place

```text
  # terraform_data.config will be updated in-place
  ~ resource "terraform_data" "config" {
        id     = "4443e7b5-57aa-b9f2-5739-2d30db4ce34c"
      ~ input  = "v1" -> "v2"
      ~ output = "v1" -> (known after apply)
    }
```

こちらは `id` がそのまま。実体を残したまま属性だけ変わります。止まりません。`~` と `-/+` の違いがすべてです。

### - destroy には理由が書いてある

```text
  # random_string.token will be destroyed
  # (because random_string.token is not in configuration)
  - resource "random_string" "token" {
      - id     = "cG!W$Xz7" -> null
      - length = 8 -> null
    }
```

`because ... is not in configuration`。HCL から消えたから削除する、と言っています。消したつもりがないなら、それは誤削除です。AI がリファクタリングのつもりでリソースブロックを消したり、名前を変えたりすると、この形で出ます(名前の変更は「古い名前を削除、新しい名前を作成」になります)。

### (known after apply)

```text
  # random_integer.port will be created
  + resource "random_integer" "port" {
      + id     = (known after apply)
      + max    = 65535
      + min    = 1024
      + result = (known after apply)
    }
```

「実行するまで値が決まらない」の意味です。ID や IP がこれで、それを他のリソースが参照していると、参照している側も作り直しになることがあります。

## 設計意図: なぜ作り直しになるのか

![なぜ -/+ か](/images/plan-symbols/why-replace.png)

クラウド側で、作った後に変更できない属性があるからです。名前、リージョン、インスタンスの種別、サブネットなど。API が更新を拒否するので、Terraform は「消して作る」しかない。それが `-/+` です。

Terraform が悪いわけではなく、クラウドの制約をそのまま見せています。問題は、消える実体が何かです。

- DB インスタンス → データ
- ディスク → 中身
- 固定 IP → 接続先
- DNS レコード → 名前解決

こういうものが `-/+` に入っていたら、「更新」ではありません。

## AI を止めるサイン

![サイン 1](/images/plan-symbols/sign-1.png)

| # | AI がやろうとしていること | 何が起きるか | 聞くこと |
|---|---|---|---|
| 1 | `-/+`(replace)を「更新されます」と要約する | DB・ディスク・固定 IP・DNS の replace は、データや接続先が消える | 「replace されるリソースはどれ? 中身は残る?」 |
| 2 | `Plan: 2 to add, 1 to change, 2 to destroy` の数字だけ見て承認する | destroy に数えられた 1 つが本番 DB かもしれない | 「destroy に数えられているのはどれ?」 |
| 3 | `prevent_destroy` のエラーを、lifecycle ブロックを消して回避する | 保険を外して通すのは事故を隠しているだけ | 「なぜその保険が要らなくなった?」 |

### サイン 1 について

AI の要約で一番多い言い換えがこれです。`-/+` は `~` の一種ではありません。`must be replaced` の行を数えて、それぞれの実体が消えてよいか 1 つずつ確認してください。

### サイン 2 について

`Plan:` の要約行で、`-/+` は「1 to destroy」と「1 to add」の両方に数えられます。つまり `2 to add, 1 to change, 2 to destroy` という数字からは、純粋な削除が 2 つなのか、replace が 2 つなのか、その混合なのかが分かりません。数字は目安で、本文を読むしかありません。

### サイン 3 について

`lifecycle { prevent_destroy = true }` を付けたリソースは、`-/+` や `-` が出た時点で plan がエラーになります。消してはいけないものに掛ける保険です。AI がこのエラーに対して lifecycle ブロックを消して通そうとしたら、保険を外しているだけで、消える事実は変わりません。「なぜ今その保険が要らなくなったのか」を聞いてください。

## まとめ

- `+` 作成 / `~` その場で更新 / `-` 削除 / `-/+` は一度消える
- `# forces replacement` で、作り直しの原因になった属性が分かる
- `-/+` が出たら、そのリソースの中身が消えてよいか確認してから承認

## シリーズ

1. [terraform init は何をしているのか](https://zenn.dev/e8dev/articles/terraform-init-ai-review)
2. [terraform plan は何を比べているのか](https://zenn.dev/e8dev/articles/terraform-plan-ai-review)
3. plan の記号を読む: `-/+` は一度消える(この記事)
4. terraform apply で何が起きるのか
5. state とは何か: Terraform の「台帳」
6. remote state と lock: 台帳を共有する

次は `terraform apply`。yes と打ったあと、何が起きているか説明できますか?
