class: center middle
# Ownership of Rust
### Rust of Us - Chapter 3
### @tanaka51

---

class: middle

このスライドは [Rust of Us - Chapter 3](https://rust-of-us.doorkeeper.jp/events/28471) での発表に使われたものに加筆修正したものです。
[Ownership](https://doc.rust-lang.org/nightly/book/ownership.html) と [References and Borrowing](https://doc.rust-lang.org/nightly/book/references-and-borrowing.html) を元に作成しました。

---

# ３部作

Rust はメモリーセーフであるために、避けては通れない、ユニークで独立した3つの機能をもっている:

- ownership
- borrowing
- lifetimes

これらをしっかり理解することが大切である。

---

# まえがき（Meta 意訳）

> Rust は安全性とスピードを重視しています。たくさんの 'zero-cost abstranctions'(Rust においては、
実際には問題にならない程小さいコストでの抽象化を意味する) を通してこれらのゴールを達成します。
ownership システムは zero-cost abstractions のうちの主軸となる例の一つです。"done at compile time"
といい、これらの機能を使うのに、実行時のコストを払う必要は全くありません。

> しかしながら、このシステムには確実にかかるコストがあります。学習曲線です。新しく Rust を始める人は、
私達でいうところの "borrow checker との戦い" を経験しています。正しいと思って書いたコードを
Rust のコンパイラがコンパイルできない、という事です。これは、だいたいは、プログラマーが考えている
ownership のメンタルモデルと、Rust に実際に実装されているの本当のルールと異なっているためです。
あなたも最初は同じようなことが経験するでしょう。でも良いニュースがあります: たくさんの Rust 熟練者
は、一度 ownership のルールを理解してしまえば、borrow checker と戦うことは本当に少なくなると言っています。

---


class: center middle
# Ownership

---

name: summary-ownership

# Ownership 3行で

.big-list[
- 参照をコピーしたら、コピー元が使えなくなる機能
  - これを Ownership が移動した(moved)という
- 関数の引数として渡しても同じく移動する
  - 戻り値で元の値を返して、呼び出し側が受け取れば Ownership も戻る
- Copy トレイトを実装した型は Ownership が **移動しない** (値をまるっとコピーするから)
]

---

# 具体例1

コンパイルエラーになる:
```rust
v = vec![1, 2, 3];

v2 = v; // ここでオーナーシップが移動する

println!("v[0] is: {}", v[0]);
```

```none
error: use of moved value: `v`
println!("v[0] is: {}", v[0]);
                        ^
```

---

# 具体例2

関数呼び出しでもコンパイルエラーになる:

```rust
fn take(v: Vec<i32>) {
    // ここで起きることは重要じゃない
}

let v = vec![1, 2, 3];

take(v); // ここでオーナーシップが移動する

println!("v[0] is: {}", v[0]); // さっきと同じエラーが発生
```

---

# 具体例3

オーナーシップは、戻り値として返せば戻せる:

```rust
fn take(v: Vec<i32>) -> Vec(i32) {
    // ここで起きることは重要じゃない

    v
}

let v = vec![1, 2, 3];

v = take(v); // ここでオーナーシップが移動して、また戻ってくる

println!("v[0] is: {}", v[0]); // エラーは発生しない
```

---

# 具体例4

Copy トレイトを実装した型(ここでは `i32`)ではエラーにならない:

```rust
let v = 1;

let v2 = v;

println!("v is: {}", v); // エラーは発生しない
```

---

# Ownership がなぜ必要か

## => セグフォを防ぐため

![image](img1.png)

この状態になると、`v` と `v2` どちらかが heap 領域を変更すると、もう片方にも影響が出てしまう。
同時にアクセスすると大変危険。

---

# コードはグチャッとしてしまう

```rust
fn foo(v1: Vec<i32>, v2: Vec<i32>) -> (Vec<i32>, Vec<i32>, i32) {
    // v1 と v2 に何かする

    // オーナーシップを戻しつつ、関数の戻り値も返す
    (v1, v2, 42)
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let (v1, v2, answer) = foo(v1, v2);
```

戻り値の宣言とか、戻り値の行とか、受け取りのところとか、複雑になってしまう。

## => Rust は borrowing という仕組みで簡潔にかけるようにした

---

template: summary-ownership

煩雑になる部分は、borrowing という仕組みで解決される。

---

class: center middle
# Borrowing

---

name: summary-borrowing

# Borrowing を三行で

.big-list[
- データレースを防ぐための機能
- 貸し出し元(Owner)より小さいスコープでのみ使える
- 参照のみと、変更可能な borrowing がある
]

---

# 参照のみの borrowing:

```rust
fn foo(v1: &Vec<i32>, v2: &Vec<i32>) -> i32 {
    // v1 と v2 に何かする

    // 答えを返す
    42
}

let v1 = vec![1, 2, 3];
let v2 = vec![1, 2, 3];

let answer = foo(&v1, &v2);

// v1 と v2 が使える!!
println!("v1[0] is {}", v1[0]);
println!("v2[0] is {}", v2[0]);
```

引数宣言に `&Vec<i32>` のように & を追加し、渡す時も & をつける

---

# 変更可能な borrowing:

```rust
let mut x = 5;
{
    let y = &mut x;
    *y += 1;
}
println!("{}", x);
```

`&mut` をつけると変更可能な貸し出しができる。
※ここでスコープをつくっているのは、オーナーよりも狭いスコープでしか Borrowing できないため。後述。

---

# Borrowing のルール

- オーナーより小さいスコープであること
- 以下のどちらかの種類の Borrowing の一つだけを持てる。同時にはもてない
  - 一つかそれ以上の、リソースへの参照(`&T`)
  - 確実に一つだけの、変更可能なリソースへの参照(`&mut T`)

2つは似ているが、データレースの定義から見ると違いがわかる。

> データレースは、非同期に、2つかそれ以上のポインターが同じ場所のメモリーを指し示し、
> それらのうちのどれか一つが書きこんでいる時に発生する。

ただの参照は書き込みが発生しないのでデータレースが起こらない。一方、変更可能な参照は
書き込みが発生するので、同時に2つ以上の取得を禁止している。これが Rust のコンパイル時の
データレースの防ぎ方で、このルールをやぶるとコンパイルエラーとなる。

---

# Borrowing とスコープの具体例1

以下はエラーとなる:

```rust
let mut x = 5;

let y = &mut x;    // -+ &mut borrow of x starts here
                   //  |
*y += 1;           //  |
                   //  |
println!("{}", x); // -+ - try to borrow x here
                   // -+ &mut borrow of x ends here
```

```console
error: cannot borrow `x` as immutable because it is also borrowed as mutable
    println!("{}", x);
                   ^
```

`&mut` は同時に一つしか存在できないのに、`y` に貸したあとに更に `println!` に貸そうとするため。

---

# Borrowing とスコープの具体例2

以下のようにすればエラーにならない:

```rust
let mut x = 5;

{
    let y = &mut x; // -+ &mut borrow starts here
    *y += 1;        //  |
}                   // -+ ... and ends here

println!("{}", x);  // <- try to borrow x here
```

`y` はスコープが終了したときに開放されるため、`println!` で貸すことができる。

---

# Borrowing が防げる課題

データレースを防ぐために Borrowing があるが、具体的には以下のような問題を未然に防げる:

- イテレータでの不正(Iterator Invalidation)
- 開放後での使用(use after free)

---

# イテレータでの不正(Iterator Invalidation)

動くコード:

```rust
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
}
```

エラーになるコード:

```rust
let mut v = vec![1, 2, 3];

for i in &v {
    println!("{}", i);
    v.push(34);
}
```

これができてしまうと、ループの最中に配列の中身が増えてループが終わらない。

=> Rust では、v はすでに for に貸し出されてるのでエラーとなる

---

# 開放後での使用1(use after free)

参照は、参照元より長く生存できない。

以下はエラーになる:

```rust
let y: &i32;
{
    let x = 5;
    y = &x;
}

println!("{}", y);
```

y は x が開放された後のメモリーを参照しようとしていて、それができてしまうと意図しない動作となる。

=> Rust では y は x が存在している間でしか存在できないのでエラー。


---

# 開放後での使用2(use after free)

同じような問題で、参照が参照元より先に宣言されていてはいけない。
同一スコープ内で宣言された変数が開放される順番が、宣言と逆順になるから。

エラーになる:

```rust
let y: &i32;
let x = 5;
y = &x;

println!("{}", y);
```

=> 開放の順番的に、y が x より長く存在していることになってしまうのでエラー。

---

template: summary-borrowing

@katsuyoshi の発表に続きます
