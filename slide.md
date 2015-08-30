class: center middle
# Ownership of Rust
### Rust of Us - Chapter 3
### @tanaka51

---

class: middle

このスライドは [Rust of Us - Chapter 3](https://rust-of-us.doorkeeper.jp/events/28471) での発表に使われたものに加筆修正したものです。
8/30 時点での [Ownership](https://doc.rust-lang.org/nightly/book/ownership.html) と [References and Borrowing](https://doc.rust-lang.org/nightly/book/references-and-borrowing.html) と [Lifetimes](https://doc.rust-lang.org/nightly/book/lifetimes.html#lifetime-elision) を元に作成しました。

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
- ある参照を他の変数に代入したら、元の変数の参照が使えなくなる機能
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

# ちょっと複雑な例

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

=> Rust の borrowing という仕組みを使うと簡潔にかける

---

template: summary-ownership

煩雑になる部分は、borrowing という仕組みを使うと簡潔になる

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

---

class: center middle
# Lifetimes

---

name: summary-lifetimes

# Lifetimes を3行で

.big-list[
- 参照のスコープを決定する仕組み
- 参照でないものは関係ない
- シグネイチャー部分での省略ルールがある
  - 覚えておくとコンパイルエラーに出くわした時に解決しやすくなる、かも
]

---

他の誰かが持っている参照を貸し出すのは、複雑になりやすい。例えば、

1. Aさんがあるリソースを取得する
1. AさんがBさんにリソースへの参照を貸し出す
1. Aさんがリソースを使い終わったので開放する、ただしBさんはまだ参照を持っている
1. Bさんがリソースを使おうとする

- Bさんのリソースは不正な場所を参照することになる
- dangling pointor とか use after free と呼ぶ
- ステップ3の後にステップ4が行われないようにしなければならない

=> Rust の Ownership は lifetimes という、参照が正しく行われるスコープを宣言するシステムで解決する

---

# 具体的な記述例

参照を引数としてとる関数は、参照への lifetime を暗黙的にも、明示的に書くこともでる:

```rust
// 暗黙的
fn foo(x: &i32) {
}

// 明示的
fn bar<'a>(x: &'a i32) {
}
```

`'a` は「ライフタイム a」と読む。

参照はそれぞれの自身に紐づくライフタイムを持つが、だいたいのケースではコンパイラが省かせてくれる。

---

# ライフタイムの記述を掘り下げる

関数宣言部分では、`<>` に囲まれた部分がライムタイムの宣言場所となる:

```rust
fn bar<'a>(...) // ひとつのライフタイム a をもった関数 bar
fn baz<'a, 'b>(...) // ふたつのライフタイム a と b をもった関数 baz
```

引数リストに、関数宣言で名前をつけたライフタイムを使うことができる:

```rust
...(x: &'a i32)
...(x: &'a mut i32) // mutable な参照が欲しければこうかく
```

---

# Struct

Struct で参照を扱うときには、明示的にライフタイムを指定する必要がある:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5; // this is the same as `let _y = 5; let y = &_y;`
    let f = Foo { x: y };

    println!("{}", f.x);
}
```

`struct` もライフタイムを持つことができる。関数と同じよう感じで宣言したり使えたりできる:

```rust
struct Foo<'a> { // 宣言
```

```rust
x: &'a i32, // 使う
```

なぜここでライフタイムが必要かというと、`Foo` の持つ参照が、`i32` 自身のスコープより長く存在できないことを確実に伝えるためである。

---

# Imple block

`Foo` にメソッドを実装する:

```rust
struct Foo<'a> {
    x: &'a i32,
}

impl<'a> Foo<'a> {
    fn x(&self) -> &'a i32 { self.x }
}

fn main() {
    let y = &5; // this is the same as `let _y = 5; let y = &_y;`
    let f = Foo { x: y };

    println!("x is: {}", f.x());
}
```

`Foo` のために、`imple` にもライフタイムを宣言する必要がある。`'a` を繰り返しているが、これは関数定義と同じように考えればよい:
`imple<'a>` がライフタイム `'a` を定義し、それを `Foo<'a>` で使う。

---

# Multiple lifetimes

たくさんの参照があれば、それに対して同じライフタイムを何度も使うことができる:

```rust
fn x_or_y<'a>(x: &'a str, y: &'a str) -> &'a str {
```

`x` と `y` と戻り値が同じスコープであることを定義している。

`x` と `y` が違うスコープなら、複数のライフタイムを使うことができる:

```rust
fn x_or_y<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
```

`x` と `y` が違うライフタイムで、戻り値が `x` と同じライフタイムになる。

---

# Thinking in scopes

スコープを視覚化して考えてみる:

```rust
fn main() {
    let y = &5;     // -+ y goes into scope
                    //  |
    // stuff        //  |
                    //  |
}                   // -+ y goes out of scope
```

これに先ほどの `Foo` を足してみる:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5;           // -+ y goes into scope
    let f = Foo { x: y }; // -+ f goes into scope
    // stuff              //  |
                          //  |
}                         // -+ f and y go out of scope
```

`f` は `y` と一緒のスコープなので動く。

---

# Thinking in scopes 2

が、こうすると動かない:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let x;                    // -+ x goes into scope
                              //  |
    {                         //  |
        let y = &5;           // ---+ y goes into scope
        let f = Foo { x: y }; // ---+ f goes into scope
        x = &f.x;             //  | | error here
    }                         // ---+ f and y go out of scope
                              //  |
    println!("{}", x);        //  |
}                             // -+ x goes out of scope
```

`f` と `y` のスコープは `x` のスコープよりも小さい。
それなのに、`x = &f.x;` として、`x` にスコープ外の参照を渡してしまっている。

---

# 'static

'static' と名付けられたライフタイムは特別なライフタイムである。プログラム全体で生き続ける。

最初に出会う `'static` は文字列であろう:

```rust
let x: &'static str = "Hello, world!";
```

文字列リテラルはずっと生きているので、型が `&'static` となる。

他にはグローバル変数がある:

```rust
static FOO: i32 = 5;
let x: &'static i32 = &FOO;
```

`i32` をデータセグメントに追加していて、`x` はそれへの参照である。

---

# Lifetime Elision

型に関しては、シグネイチャー部分での推論はできない（省略できない）。けれども、ライフタイムに関しては
限定的であるが推論されるようになっている。省略には3つのルールがある。

ルールの説明の前に、用語をおさえておく。*input lifetime* は仮引数リストで使われるライフタイムで、*output lifetime* は戻り値で使われるライフタイムである。

以下がルールである:

- 関数の仮引数リストで省略された場合、それぞれ独立したライフタイムを持つ
- 引数がちょうど一つだけの場合、省略されてようがされてなかろうが、そのライフタイムがすべての省略された関数の戻り値に割り当てられる。
- 複数の input lifetime があり、そのうちの一つが `&self` か `&mut self` だった場合、`self` のライフタイムが、すべての省略された output liftime に割り当てられる

言い換えると、output lifetime が決定できない状況だとエラーになるということである。

---

# Examples

```rust
fn print(s: &str); // 省略したもの
fn print<'a>(s: &'a str); // 展開したもの

fn debug(lvl: u32, s: &str); // 省略したもの
fn debug<'a>(lvl: u32, s: &'a str); // 展開したもの

// 直前のサンプルでの `lvl` にはライフタイムは必要ない。なぜなら参照(`&`)ではないから。
// 参照に関係するものだけにライフタイムは必要となる(参照をもった `struct` とか)

fn substr(s: &str, until: u32) -> &str; // 省略したもの
fn substr<'a>(s: &'a str, until: u32) -> &'a str; // 展開したもの

fn get_str() -> &str; // input がないものは不正となる。コンパイルエラー

fn frob(s: &str, t: &str) -> &str; // 2つの input があり、戻り値のライフタイムを特定できないのでコンパイルエラー
fn frob<'a, 'b>(s: &'a str, t: &'b str) -> &str; // 展開してみるとわかる

fn get_mut(&mut self) -> &mut T; // 省略したもの
fn get_mut<'a>(&'a mut self) -> &'a mut T; // 展開したもの

fn args<T:ToCStr>(&mut self, args: &[T]) -> &mut Command // 省略したもの
fn args<'a, 'b, T:ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // 展開したもの(`self` のライフタイムが割り当てられる)

fn new(buf: &mut [u8]) -> BufWriter; // 省略したもの
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a> // 展開したもの
```

---

template: summary-lifetimes

---

# 全体的にまとめ

#### Ownership

- セグメンテーションフォルトを起こさせないための仕組み
- あるリソースへの参照は、オーナシップを持った変数だけがアクセスできる
- 代入や関数への引数渡しによってオーナーシップが移動する

#### Borrowing

- データレースを起こさせないための仕組み
- オーナーシップを移動させずに参照だけを渡すことができる
- オーナーより小さいスコープでのみ存在できる
- mutable な borrowing は必ず一つだけしか存在しないことが保証される

#### Lifetimes

- Borrowing の上位概念(データレースを起こさせない的な意味で)
- あるリソースへの参照のスコープを決定する仕組み
