---
title: "ガード節の使い方"
emoji: "🤭"
type: "tech"
topics:
  - "rails"
  - "ruby"
published: true
published_at: "2023-07-02 08:56"
---

### ガード節 とは？

**条件を確認し、条件が満たされた場合に早期にメソッドを終了するためのテクニック**です。

ガード節を使用すると、条件が満たされた場合に return により メソッドをすぐに終了させ、条件以降の処理を実行されないようにします。

これにより、不要なコードの実行を避け、コードの可読性と保守性を向上させることができます。

ガード節は、引数の妥当性の検証、データの存在確認、ある条件による処理の制御、ループの終了条件などがコード内にある時に使用されます。

（コード内にネストが深いコードがあるときも、ガード節を用いるとリファクタリングできるかもしれません。）

使用例は以下のとおりです。

```ruby
def method_name(arguments)
  return unless condition

  # 条件が満たされた場合の処理
end
```

**`return unless condition`** の行がガード節であり、**`condition`** が満たされた場合（上記の場合、condition が false の時）にメソッドが終了します。

また、ガード節は複数使用することもできます。

条件を追加するために複数のガード節を使用することで、より複雑な条件を表現できます。

使用例は以下のとおりです。

```ruby
# 複数のガード節
def method_name(arguments)
  return unless condition1
  return unless condition2

  # 条件が満たされた場合の処理
end
```

さらに、ガード節を使用すると、条件だけでなく、引数の妥当性を検証する際にも役立ちます。

```ruby
def save_user(user_params)
  return unless user_params[:name].present?
  return unless user_params[:email].present?

  # ユーザーを保存する処理
end
```

上記の例では、**`user_params`** の **`name`** と **`email`** の値が存在しない場合に早期リターンし、ユーザーを保存する処理を行いません。

以上が、ガード節の使い方の説明になります。




&nbsp;
 ### 参考

https://github.com/fortissimo1997/ruby-style-guide/blob/japanese/README.ja.md#no-nested-conditionals

https://ichi-station.com/%E3%82%AC%E3%83%BC%E3%83%89%E7%AF%80%E3%81%AB%E9%96%A2%E3%81%97%E3%81%A6/

https://techracho.bpsinc.jp/hachi8833/2016_10_11/26950