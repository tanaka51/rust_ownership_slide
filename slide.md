class: center middle
# Ownership of Rust
### Rust of Us - Chapter 3
### @tanaka51

---

# ３部作

.big-big-list[
- Ownership
- Borrowing
- lifetimes
]

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
- オーナーシップを移動せずに、貸し出す機能
- 貸し出し元(Owner)より小さいスコープでのみ使える
- 参照のみの貸し出しと、変更可能な貸し出しがある
]

---

# 具体例1

参照のみの貸し出し:

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
```

引数として `&Vec<i32>` のように & を追加するだけ

---

# 具体例2

変更可能な貸し出し:

```rust
let mut x = 5;
{
    let y = &mut x;
    *y += 1;
}
println!("{}", x);
```

`&mut` をつけると変更可能な貸し出しができる。
ここでスコープをつくっているのは、オーナーよりも狭いスコープでしか Borrowing できないため。

---

# Borrowing とスコープ

```rust
let mut x = 5;

let y = &mut x;    // -+ &mut borrow of x starts here
                   //  |
*y += 1;           //  |
                   //  |
println!("{}", x); // -+ - try to borrow x here
                   // -+ &mut borrow of x ends here
```

```rust
let mut x = 5;

{
    let y = &mut x; // -+ &mut borrow starts here
    *y += 1;        //  |
}                   // -+ ... and ends here

println!("{}", x);  // <- try to borrow x here
```

---

# なぜこんなルールがあるのか？

## => データレースを防ぐため

どんな問題を防げるのか

---

# イテレータでの不正を防ぐ

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

=> これができないのは、v はすでに for に貸し出されてるから

---

# 開放後へのアクセスを防ぐ1

エラーになる:

```rust
let y: &i32;
{
    let x = 5;
    y = &x;
}

println!("{}", y);
```
=> y は x が存在している間でしか存在できないのでエラー。

---

# 開放後へのアクセスを防ぐ2

エラーになる:

```rust
let y: &i32;
let x = 5;
y = &x;

println!("{}", y);
```

=> y が x より長く存在していることになってしまうのでエラー。

---

template: summary-borrowing

@katsuyoshi の発表に続きます
