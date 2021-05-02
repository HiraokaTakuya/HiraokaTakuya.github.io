---
layout: default
---

# Rustで大きな構造体を確保する際のスタックオーバーフローを回避する方法

Rust では大きな構造体や配列を確保する時、少し工夫が必要です。<br>
例えば以下のコードをデバッグモードでコンパイルすると、実行時にエラーになります。

```rust
fn main() {
    let _a = [1_i8; 100_000_000];
}
```
```bash
cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/sample`

thread 'main' has overflowed its stack
fatal runtime error: stack overflow
中止 (コアダンプ)
```

これがスタックオーバーフローです。<br>
プログラムは実行時に最初に一定の大きさのスタックを確保します。<br>
Rust では 2MB 確保されます。<br>
(C++ などでは OS やコンパイラ毎にデフォルト値が設定されていて、リンク時に設定する事でスタックのサイズを変更する事が出来ます。)<br>
スタックはプログラムの動作中、変数を確保した時などに消費され、<br>
変数がスコープを抜けた時に開放されます。<br>
プログラム全体でサイズが決まっていて不便ですが、スタックの消費と解放は、単にスタックのポインタを変更するだけなので、非常に高速に動作します。<br>
<br>
プログラム中に大きなデータを確保するには、ヒープを利用すると良いです。<br>
ヒープは基本的にプログラムが動作する環境のメモリに空きがあるだけ使えます。<br>
先程のサンプルをヒープを使ったプログラムに直すには、丁度 Vec を使えば良いです。

```rust
fn main() {
    let _a = vec![1_i8; 100_000_000];
}
```

簡単ですね。<br>
もう少し複雑な例として、配列を持った struct が入れ子になった場合を考えます。<br>

## 最初のコード (スタックオーバーフローする）

```rust
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    let _c = C::new(B::new(A::new(1)));
}
```
- A は i8 を 512 個持つので、512 bytes
- B は A を 512 個持つので、512 * 512 = 262144 bytes
- C は B を 512 個持つので、512 * 512 * 512 = 134217728 bytes

であり、C をスタックに確保するとスタックオーバーフローになります。<br>

## フィールドを配列から Vec に変更する方法
これを Vec を使う事でスタックオーバーフローを回避するように変更します。

```rust
const N: usize = 512;

#[derive(Debug, Clone)]
struct X<T: Clone>(Vec<T>);
impl<T: Clone> X<T> {
    fn new(item: T) -> Self {
        Self(vec![item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    let _c = C::new(B::new(A::new(1)));
}
```
Vec に変更した事で、要求する trait 境界が Copy から Clone になりましたが、少ない変更でスタックオーバーフローを回避できました。<br>
<br>
このように配列を Vec に変更する事で全てが解決すれば良いのですが、<br>
複数の小さい Vec の集まりは、順にアクセスする際にキャッシュヒットしにくく、アクセス速度の低下に繋がります。<br>
ボトルネックでない箇所であれば問題ありませんが、常にそうとは限らないので、Vec に変更しないで配列のままスタックオーバーフローを回避する方法も知っておきたいです。<br>
<br>

## Box::new() を使う方法
```rust
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    let _c = Box::new(C::new(B::new(A::new(1))));
}
```
Box::new() は引数のデータをヒープに確保します。<br>
しかしながら、これはスタックオーバーフローになってしまいます。<br>
理由は、Box::new() に引数として渡す C::new(B::new(A::new(1))) が一時的にスタックに確保されるためです。<br>
これが出来れば全く苦労はしないのですが、もう少し工夫する必要があります。<br>

## Box::new_uninit() を使う方法
```rust
#![feature(new_uninit)]
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    let _c = {
        // Box::new_uninit() is a nightly-only API.
        let mut c = Box::<C>::new_uninit();
        unsafe {
            let c_ptr = c.as_mut_ptr();
            for b in &mut (*c_ptr).0 {
                for a in &mut b.0 {
                    *a = A::new(1);
                }
            }
            c.assume_init()
        }
    };
}
```
これは nightly コンパイラで、
```rust
#![feature(new_uninit)]
```
を付ける事で使用することが出来ます。
```rust
let mut c = Box::<C>::new_uninit();
```
C を未初期化でヒープに領域を確保した後、ポインタ経由で値を設定していきます。<br>
この方法では C::new() が使えない為、C よりも複雑な初期化を行う型では、値を設定する所で間違わないようにする必要があります。

## ポインタにヒープ領域を確保してBoxで返す方法
```rust
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    let _c = unsafe {
        let c_ptr = std::alloc::alloc(std::alloc::Layout::new::<C>()) as *mut C;
        for b in &mut (*c_ptr).0 {
            for a in &mut b.0 {
                *a = A::new(1);
            }
        }
        Box::from_raw(c_ptr)
    };
}
```
ポインタに未初期化のヒープ領域を確保し、値を設定していきます。<br>
これは stable コンパイラで使用出来ます。<br>
Box::new_uninit() を使う時と同様に、この方法では C::new() が使えない為、C よりも複雑な初期化を行う型では、値を設定する所で間違わないようにする必要があります。<br>
<br>
## 大きなスタックのスレッドを生成する方法
```rust
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn main() {
    const STACK_SIZE: usize = 512 * 1024 * 1024;
    let _c = std::thread::Builder::new()
        .stack_size(STACK_SIZE)
        .spawn(|| Box::new(C::new(B::new(A::new(1)))))
        .unwrap()
        .join()
        .unwrap();
}
```
Rust ではプログラム実行時のスタックサイズは 2MB (変更可能かはちょっと知らないです。)ですが、<br>
生成するスレッドのスタックサイズは自由に決められます。<br>
このコードでは Box\<C\> を返したら即座にスレッドを終了しています。<br>
C の領域確保にスレッド生成と破棄の時間を掛けて良いなら、これで良いでしょう。<br>
しかし、次の例の様にプログラム全体を大きなスタックのスレッドで動かせば、<br>
他の部分を特に変更する事無く、C の領域確保を含めプログラム全体が問題なく動作します。

## プログラム全体を大きなスタックのスレッドで動作させる方法
```rust
const N: usize = 512;

#[derive(Debug, Clone, Copy)]
struct X<T: Copy>([T; N]);
impl<T: Copy> X<T> {
    fn new(item: T) -> Self {
        Self([item; N])
    }
}
type A = X<i8>;
type B = X<A>;
type C = X<B>;

fn sub_main() {
    let _c = C::new(B::new(A::new(1)));
    let _c = Box::new(C::new(B::new(A::new(1))));
}

fn main() {
    const STACK_SIZE: usize = 512 * 1024 * 1024;
    std::thread::Builder::new()
        .stack_size(STACK_SIZE)
        .spawn(sub_main)
        .unwrap()
        .join()
        .unwrap();
}
```
ここでは main 関数で大きなスタックのスレッドを生成し、その中で実質的にプログラムを記述します。<br>
スタックサイズさえ大きければ、最初に例に出した単に C::new() を呼ぶ方法が使えるようになります。<br>
もちろん Box を使っても大丈夫です。<br>
これが一番手軽な方法と言えるかも知れませんが、ライブラリを書く際には使えない事と、<br>
プログラム開始直後に大きなスタックサイズの分だけメモリを消費する事だけ注意して下さい。<br>

## まとめ

- 基本的には main 関数で大きなスタックのスレッドを生成すれば良い。
- それが出来ない時、ポインタを使ったヒープ領域の確保も有効。
- ポインタを使ったヒープ領域の確保では、確保した変数に値を設定する所を間違わないようにしなければならない。

他にも良い方法があれば、サンプルコードのリポジトリの issue 等で教えて頂けると助かります。

[サンプルコードのリポジトリ](https://github.com/HiraokaTakuya/some_samples_of_how_to_avoid_stack_overflow_when_allocating_large_struct_in_rust)<br>
