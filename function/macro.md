# 巨集

## 簡介

[crate std中的macro](https://doc.rust-lang.org/std/index.html#macros)

“巨集”（macro）是Rust的一個重要特性。Rust的“巨集”（macro）是一種編譯器擴展，它的調用方式為`some_macro！（...）`。巨集調用與普通函式呼叫的區別可以一眼區分開來，凡是巨集調用後面都跟著一個驚嘆號。

巨集也可以通過`some_macro！[...]`和`some_macro！{...}`兩種語法調用，只要括弧能正確匹配即可。

```rust
fn main(){
    println!("hello world 1");
    println!["hello world 2"];
    println!{"hello world 3"};
}
```

* 首先，Rust的巨集在調用的時候跟函數有明顯的語法區別；
* 其次，<mark style="color:blue;">巨集的內部實現和外部調用者處於不同名字空間</mark>，它的訪問範圍嚴格受限，是通過參數傳遞進去的，我們不能隨意在巨集內訪問和改變外部的程式碼。
* C/C++中的巨集只在預處理階段起作用，因此只能實現類似文本替換的功能。<mark style="color:blue;">而Rust中的巨集在語法解析之後起作用，因此可以獲取更多的上下文資訊，而且更加安全</mark>。

## 實現編譯階段檢查

使用巨集，我們可以在編譯階段分析這個字串常量和對應參數，確保它符合約定。另外一個常見的場景是，利用巨集來檢查規則運算式的正確性。

```rust
fn main() {
    // error: 2 positional arguments in format string, 
    // but no arguments were given
    println!("number1 {} number2 {}");
}
```

## 實現編譯期計算

[Macro std::file](https://doc.rust-lang.org/std/macro.file.html)

[Macro std::line](https://doc.rust-lang.org/std/macro.line.html)

```rust
// 可以列印出當前原始程式碼的檔案名，
// 以及當前程式碼的行數。這些資訊都是純編譯階段的資訊。
fn main() {
    // file src/main.rs line 4 
    println!("file {} line {} ", file!(), line!());
}
```

## 實現自動程式碼生成

有些情況下，許多程式碼具有同樣的“模式”，但是它們不能用現有的語法工具，如“函數”“泛型”“trait”等對其進行合理抽象。那麼我們可以用“巨集”來精簡程式碼，消除重複。

比如，在標準庫中就有許多類似的用法。在core/ops.rs程式碼中，內置類型對各種運算子trait的支援就使用了巨集。

```rust
add_impl! { usize u8 u16 u32 u64 isize i8 i16 i32 i64 f32 f64 }
```

## 實現語法擴展

某些情況下，我們可以使用宏來設計比較方便的“語法糖”，而不必使用編譯器內部硬編碼來實現。比如初始化一個動態陣列，我們可以使用方便的vec！巨集：

[Macro std::vec](https://doc.rust-lang.org/std/macro.vec.html)

```rust
let v = vec![1, 2, 3, 4, 5];
```

我們可以充分發揮自己的想像力，通過自訂巨集來增加語言的表達能力，甚至自訂DSL（Domain Specific Language）。

## 自定義巨集

自訂巨集有兩種實現方式：

* 標準庫提供的macro\_rules！巨集（範例巨集）實現。
*  通過提供編譯器擴展來實現。

編譯器擴展只能在不穩定版本中使用。它的API正在重新設計中，還沒有正式定稿，這就是所謂的macro 2.0。

## 範例巨集

macro\_rules！是標準庫中為我們提供的一個編寫簡單巨集的小工具，它本身也是用編譯器擴展來實現的。它可以提供一種“示範型”（by example）巨集編寫方式。

以下範例提供一個hashmap！巨集，實現如下初始化HashMap的功能：

```rust
// 在大括弧裡面，我們定義巨集的使用語法，以及它展開後的形態
macro_rules! hashmap {
    // 定義方式類似match語句的語法，expander=>{transcriber}。
    // 左邊的是巨集擴展的語法定義，後面是巨集擴展的轉換機制
    // 語法定義的識別字以$開頭，類型支援item、block、stmt、
    // pat、expr、ty、itent、path、tt。
    // 我們的需求是需要一個運算式，一個“=>”識別字，再跟一個運算式。
    
    // 現在我們希望在巨集裡面，可以支援重複多個這樣的語法元素。
    // 我們可以使用+模式和*模式來完成。類似規則運算式的概念，
    // +代表一個或者多個重複，*代表零個或者多個重複。 
    ($( $key: expr => $val: expr ),*) => {{
        let mut map = ::std::collections::HashMap::new();
        // 最後，我們在語法擴展的部分也使用*符號，將輸入部分擴展為多條insert語句。
        $( map.insert($key, $val); )*
        map
    }}
}
fn main() {
    let counts = hashmap!['A' => 0, 'C' => 0, 'G' => 0, 'T' => 0];
    println!("{:?}", counts);
}
```

一個自訂的巨集就誕生了。如果我們想檢查一下巨集展開的情況是否正確，可以使用如下rustc的內部命令：

```rust
// error: the option `Z` is only accepted on the nightly compiler
rustc -Z unstable-options --pretty=expanded temp.rs

// 從nightly-2021-07-28版本後，要改成
rustc -Zunpretty=expanded temp.rs
```

巨集展開後的內容如下：

```rust
fn main() {
    let counts = {
        let mut map = ::std::collections::HashMap::new();
        map.insert('A', 0);
        map.insert('C', 0);
        map.insert('G', 0);
        map.insert('T', 0);
        map
    };
    {
        ::std::io::_print(
            match match (&counts,) {
                (arg0,) => [::core::fmt::ArgumentV1::new(arg0, ::core::fmt::Debug::fmt)],
            } {
                ref args => unsafe { ::core::fmt::Arguments::new_v1(&["", "\n"], args) },
            },
        );
    };
}
```





## Format格式

Rust中還有一系列的巨集，都是用的同樣的格式控制規則，如format！write！writeln！等。詳細文件可以參見標準庫文件中[std::fmt](https://doc.rust-lang.org/std/fmt/index.html)模組中的說明。

```rust
fn main() {
    println!("{}", 1); // 1, 預設用法,列印Display
    println!("{:o}", 9); // 11, 八進制
    println!("{:x}", 255); // ff, 十六進位 小寫
    println!("{:X}", 255); // FF, 十六進位 大寫
    println!("{:p}", &0); // 指針
    println!("{:b}", 15); // 1111，二進位
    println!("{:e}", 10000f32); // 1e4, 科學計數(小寫)
    println!("{:E}", 10000f32); // 1E4, 科學計數(大寫)
    println!("{:?}", "test"); // 列印Debug
    println!("{:#?}", ("test1", "test2")); // 帶換行和縮進的Debug列印
    println!("{a} {b} {b}", a = "x", b = "y"); // 具名引數
}
```
