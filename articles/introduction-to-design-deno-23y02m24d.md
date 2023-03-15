---
title: "良いコード/悪いコードで学ぶ設計入門を読みました。"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true 
---

# この記事について

「良いコード/悪いコードで学ぶ設計入門」を読んだので自分なりに参考担った部分をまとめてみました。

# 役立つ設計パターンの例

| パターン                     | 効果                                                               |
| ---------------------------- | ------------------------------------------------------------------ |
| 完全コンストラクタ           | 不正状態から防護する                                               |
| 値オブジェクト               | 特定の値に関するロジックを高凝縮にする                             |
| ストラテジ                   | 条件分岐を削減し、ロジックを単純化する                             |
| ポリシー                     | 条件分岐を単純化したり、カスタマイズできるようになる               |
| ファーストクラスコレクション | 値オブジェクトの亜種で、コレクションに関するロジックを高凝縮にする |
| スプラウトクラス             | 既存のロジックを変更せずに、安全に新機能を追加する                 |

## 完全コンストラクタ

- インスタンス変数をすべて初期化できるだけの引数を持ったコンストラクタを用意する
- コンストラクタ内では、ガード節を用いて、不正値を弾く
- インスタンス変数を不変にする(`readonly`)にすることで、生成後に不正な状態に陥らなくなる

## 値オブジェクト

値をクラスとして表現する設計パターン。\
例えば税抜金額や税込み金額を`number`型でローカル変数や引数で制御していると、金額計算ロジックがあちこちに書かれて低凝縮になる。\
また他の`number`型の「注文数」などが不注意で代入される可能性を防ぐことができる。（コンパイル時に気づける）

## 例

```typescript
class Money {
  constructor(
    // readonlyにすることで、不変にしている。
    private readonly amount: number,
    private readonly currency: string,
  ) {
    // ガード節を用いて、不正な値が代入されることを防ぐ（そのため、インスタンス化の後は不正な値があることを考えなくていい）
    if (amount < 0) {
      throw new Error("金額には0以上を指定してください。");
    }

    if (!currency) {
      throw new Error("通貨単位を指定してください。");
    }
  }

  // Moneyクラスに金額加算メソッドを用意する
  add(other: Money): Money {
    const added = this.amount + other.amount;

    if (this.currency !== other.currency) {
      throw new Error("通貨単位が違います。");
    }

    // インスタンスを不変にするため、新しく作成したインスタンスを返却するようにしている。
    return new Money(added, this.currency);
  }
}
```

# 不変の活用

変数に再度値を代入することを再代入(破壊的代入)と呼ぶ。\
変数の意味が変わってしまうため、読みづらく、変更を負うのが難しくなってしまう。

typescriptでは、`readonly`や`const`、オブジェクトの場合は型に`Readonly`をつけることで、不変であるように扱える。\
不変にすると以下のメリットを得ることができる。

- 変数の意味が変化しないので、混乱しない。
- 挙動が安定し、結果を予測しやすくなる
- コードの影響範囲が限定的になり、保守が容易になる

# 凝縮度

モジュール内における、データとロジックの関係性の強さを表す指標。\
高凝縮な構造は変更に強く、低凝縮な構造は壊れやすい。

以下のようなケースは一見良さそうですが、低凝縮な構造になってしまっています。(データとロジックが別のクラスに定義されているため)

```typescript
class OrderManager {
  static add(money1: number, money2: number) {
    return money1 + money2;
  }
}
```

可能であれば以下のようにまとめると、高凝縮になる。

```typescript
class Order {
  constructor(private readonly amount: number) {}

  add(target: Order): Order {
    return new Order(this.amount + target.amount);
  }
}
```

# ファクトリメソッド

コンストラクタを公開していると、さまざまな用途に使われてしまい、ロジックが分散してしまうことがある。\
その場合は、コンストラクタをprivateにして、目的別のファクトリメソッドを用意することでロジックをクラス内に凝縮することができる。

```typescript
class Order {
  private static readonly PREMIUM_MEMBER_DISCOUNT_RATE = 0.95;
  private static readonly STANDARD_MEMBER_DISCOUNT_RATE = 1;
  // privateにすることで、外部からインスタンス化できないようにする
  private constructor(private readonly amount: number) {}

  // ファクトリメソッド1: 通常会員の注文の場合
  static standardUserOrder(amount: number) {
    return new Order(amount * Order.STANDARD_MEMBER_DISCOUNT_RATE);
  }

  // ファクトリメソッド1: プレミアム会員の注文の場合
  static premiumUserOrder(amount: number) {
    return new Order(amount * Order.PREMIUM_MEMBER_DISCOUNT_RATE);
  }
}
```

# DRY原則の誤用/過度な共通化

DRY原則は重複を避けるという意味ではなく、意味が重複した重複した表現を避けるという考え方のほうがいい。\
以下は割引に関するコードを載せており、discountedAmountの記述が重複しているが、通常の割引と夏の割引では仕様が違うことが考えられるので、この記述で問題ない。\
（例えば、夏の割引だけ商品の5%を割り引くことになったとき、無理に共通化していると対応することが難しくなる）

```typescript
class Price {
  constructor(private readonly amount: number) {}
}
// 通常期の割引に関するクラス
class RegularDiscountRate {
  private static readonly MIN_AMOUNT = 0;
  private static readonly DISCOUNT_AMOUNT = 1000;
  private readonly amount;
  constructor(amount: number) {
    const discountedAmount = price.amount - RegularDiscountRate.DISCOUNT_AMOUNT;
    this.amount = discountedAmount < RegularDiscountRate.MIN_AMOUNT
      ? RegularDiscountRate.MIN_AMOUNT
      : discountedAmount;
  }
}

// 夏の割引に関するクラス
class SummerDiscountRate {
  private static readonly MIN_AMOUNT = 0;
  private static readonly DISCOUNT_AMOUNT = 2000;
  private readonly amount;
  constructor(amount: number) {
    const discountedAmount = price.amount - RegularDiscountRate.DISCOUNT_AMOUNT;
    this.amount = discountedAmount < RegularDiscountRate.MIN_AMOUNT
      ? RegularDiscountRate.MIN_AMOUNT
      : discountedAmount;
  }
}
```

また、よく作成しがちな`Util`のような便利クラスを使うと、凝縮度の低いコードになってしまう。

```typescript
class Util {
  // 注文をもとに表示するテキストを生成する関数
  public getOrderStatusText(order: Order) {}
  // Workerにtaskを送信する関数
  public publishEvent(event: Event) {}
}
```

# DTO

更新する責務と、参照する責務でモジュールを分離する考え方を持ったアーキテクチャパターンがある。\
その場合、データベースの値を格納し、表示側に転送するだけのクラスとして設計することがある。

# 所感

DRY原則の部分は、自分の考え方が変わるような大きな衝撃を受けた。\
業務ではtypescriptとpythonをよく使用しているが、今までクラスを使わず、低凝縮な書き方をしてしまっていた。\
例えば注文であれば`Order`に凝縮したり、手数料なら`Fee`に凝縮するように意識をしたい。\
ただ、既存で適用されていない考え方を新しく導入するのは難しいと思うので、どのようにこの考え方を広げていくか、考える必要はあると感じた。
