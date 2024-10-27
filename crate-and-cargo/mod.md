# mod

## 簡介

<mark style="color:red;">Rust模組其實就是命名空間，</mark>用關鍵詞mod表示。它的作用是把一個crate的程式碼劃分成可管理的部分。每一個crate都有一個頂層的匿名根命名空間, 根空間下面的命名空間可以任意巢狀，這樣構成一個樹形結構。

模組內部又可以包含模組。Rust中的模組是一個典型的樹形結構。每個crate會自動產生一個跟當前crate同名的模組，作為這個樹形結構的根節點。

寫模組的目的：

1. 為了分隔邏輯塊。
2. 為了提供適當的函數，或物件供外部訪問。

<mark style="color:red;">而模組中的內容預設是私有的，只有模組內部能訪問</mark>。

<mark style="color:red;">可以不使用use關鍵字，單純使用mod與完整路徑呼叫crate內部的函數</mark>。

## 路徑有兩種形式

* <mark style="background-color:red;">絕對路徑（absolute path）</mark>是以 crate 根（root）開頭的全路徑；對於外部 crate 的程式碼，是以 crate 名開頭的絕對路徑，對於當前 crate 的程式碼，則以字面值 crate 開頭。&#x20;
* <mark style="color:red;">相對路徑（relative path）</mark>從當前模組開始，以 self、super 或定義在當前模組中的識別碼開頭。

## 從Rust編譯來理解

Rust編譯器只接受一個.rs檔案作為輸入，並且只生成一個crate。

生成的crate分兩種，原始檔中有main函數會生成可執行檔案，無main函數則生成函式庫。

```rust
// 生成可執行檔
// main.rs
fn main() {
    println!("hello, rust");
}
```

```rust
// 生成函式庫，必須加pub關鍵字才可被外部使用
// lib.rs
pub fn hello() {
    println!("hello, rust");
}
```

### mod搜尋順序

在 Rust 裡試著使用 mod 關鍵字要來宣告一個模組的時候，這會試著請編譯器試著找看看有沒有對應的檔案或模組，例如：

1. 使用 mod say\_something; 這樣寫的時候，編譯器會試著找 say\_something.rs 或是 say\_something/mod.rs 檔案。
2. &#x20;如果找到了 say\_something.rs 檔案，編譯器會把裡面的程式碼當做 say\_something 模組並引入到目前的檔案裡。&#x20;
3. 同樣的，如果找到了 say\_something/mod.rs 檔案，編譯器也會把裡面的程式碼當做 say\_something 模組引入當前這個檔案裡。

## mod 概念

模組允許你將程式碼組織到單獨的檔案中。它們將你的程式碼劃分為可以在其他模組或程式中匯入和使用的邏輯部分。簡而言之，mod 用於指定模組和子模組，以便你可以在當前的 .rs 檔案中使用它們，這可能很複雜。

使用哪種方式編寫模組取決於當時的場景。

* 如果我們需要創建一個小型子模組，比如單元測試模組，那麼直接寫到一個檔內部就非常簡單而且直觀；
* 如果一個模組內容相對有點多，那麼把它單獨寫到一個檔內是更容易維護的；
* 如果一個模組的內容太多了，那麼把它放到一個資料夾中就更合理，因為我們可以把真正的內容繼續分散到更小的子模組中，而在mod.rs中直接重新匯出（re-export）。這樣mod.rs的源碼就大幅簡化，不影響外部的調用者。

## 在內部創建新模組的方式

### 同一個檔中創建內嵌模組

```bash
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── main.rs
```

直接使用mod關鍵字即可，模組內容包含到大括弧內部：

```rust
// main.rs
// 頂層的mod aaa和fn main為同一層，可直接使用
mod aaa {
    const X: i32 = 10;
    // 必須加上pub才可被外部呼叫
    pub fn print_aaa() {
        println!("aaa");
        // bbb中的函數也要加上pub在此處才可被呼叫
        bbb::print_bbb();
    }
    // mod bbb在mod aaa內為private，
    // 不可被外部直接呼叫，只能在內部使用
    mod bbb {
        pub fn print_bbb() {
            println!("bbb");
        }
    }
}
mod ccc {
    // print_ccc在mod ccc為private
    fn print_ccc() {
        println!("{}", 25);
    }
}

fn main() {
    aaa::print_aaa(); // aaa bbb
    // aaa::bbb::print_bbb(); error, bbb為pricate
    // ccc::print_ccc();   //error print_ccc為private
}
```

### <mark style="color:red;">獨立的一個檔就是一個模組</mark>

<mark style="color:red;">檔案名即是模組名。</mark>

```bash
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── main.rs
│   ├── mylib.rs
│   └── my_nestedlib.rs

```

```rust
//main.rs
// 宣告mod後即可備使用(只能呼叫pub)
mod my_nestedlib;
mod mylib;

fn main() {
    // 必須用全名呼叫函數
    my_nestedlib::nested::nested_hello(); //hello nested lib
    mylib::hello(); // hello mylib
}

// my_nestedlib.rs
pub mod nested {
    pub fn nested_hello() {
        println!("hello nested lib");
    }
}


// mylib.rs
pub fn hello()  {
    println!("hello mylib");
}
```

### 一個資料夾也可以創建一個模組

資料夾內部要有一個mod.rs文 件，這個檔就是這個模組的入口，而資料夾名稱就是mod名稱。

如果同一資料夾內有同名的檔案與資料夾(add.rs與add directory)時，使用`mod add;`會出現錯誤。

```bash
Cargo.toml
- src
    - add/
        - add_one.rs
        - mod.rs
    - main.rs
```

```rust
// src/main.rs
mod add;

fn main() {
    print!("{}", add::add_one::add_one(0));
}

// src/add/mod.rs
pub mod add_one;

// src/add/add_one.rs
pub fn add_one (base: u32) -> u32 {
  base + 1
}
```

## 多檔案模組的層級關係

Rust 的模組支援層級結構，但這種層級結構本身與檔案系統目錄的層級結構是解耦的。

`mod xxx;` 這個 `xxx` 不能包含 `::` 號。也即在這個表達形式中，是沒法引用資料夾多層結構下的模組的。因此無法直接使用 `mod a::b:c::d;` 的形式來引用資料夾中 `a/b/c/d.rs` 這個模組。

Rust 的多層模組遵循如下兩條規則：

1. 優先尋找xxx.rs 檔案
   * main.rs、lib.rs、mod.rs中的mod xxx; <mark style="color:red;">預設優先尋找同級目錄下的 xxx.rs 檔案</mark>；
   * 其他檔案yyy.rs中的mod xxx;<mark style="color:red;">預設優先尋找同級目錄的yyy目錄下的 xxx.rs 檔案</mark>；
2. 如果 xxx.rs 不存在，則尋找 xxx/mod.rs 檔案，即 xxx 目錄下的 mod.rs 檔案。(如果同時存在xxx.rs與xxx資料夾時，使用mod xxx編譯器會報錯)。

上述兩種情況，載入成模組後，效果是相同的。Rust 就憑這兩條規則，通過迭代使用，結合 pub 關鍵字，實現了對深層目錄下模組的載入；

```bash
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── a
│   │   ├── b
│   │   │   ├── c
│   │   │   │   ├── d.rs
│   │   │   │   └── mod.rs
│   │   │   └── mod.rs
│   │   └── mod.rs
│   └── main.rs

```

```rust
// a/b/c/d.rs
pub fn print_ddd() {
    println!("i am ddd.");
}

// a/b/c/mod.rs
pub mod d;

// a/b/mod.rs
pub mod c;

// a/mod.rs 
pub mod b;

// main.rs
mod a;

fn main() {
    // 使用完整路徑呼叫函數
    a::b::c::d::print_ddd(); // i am ddd.

    // 使用use縮短路徑
    use a::b::c::d;
    d::print_ddd(); // i am ddd.
}

```

Rust 要這樣設計，有以下幾個原因：

1. Rust 本身模組的設計是與作業系統檔案系統目錄解耦的，因為 Rust 本身可用於作業系統的開發；
2. Rust 中的一個檔案內，可包含多個模組，直接將 a::b:c::d 對應到 a/b/c/d.rs 會引起一些歧義；
3. Rust 一切從安全性、顯式化立場出發，要求引用路徑中的每一個節點，都是一個有效的模組，比如上例，d 是一個有效的模組的話，那麼，要求 c, b, a 分別都是有效的模組，可單獨引用。

## pub可見性

我們可以給模組內部的元素指定可見性。預設都是私有，除了兩種例外情況：

* 一是用pub修飾的trait內部的關聯元素（associated item），預設是公開的；
* 二是pub enum內部的成員預設是公開的。
* <mark style="color:blue;">如果一個元素是私有的，那麼只有本模組內的元素以及它的子模組可以訪問</mark>；
* <mark style="color:red;">如果一個元素是公開的，那麼上一層的模組就有權訪問它</mark>。

```rust
mod top_mod1 {
    pub fn method1() {}
    pub mod inner_mod1 {
        pub fn method2() {}
        fn method3() {}
    }
    mod inner_mod2 {
        fn method4() {}
        mod inner_mod3 {
            fn call_fn_inside() {
                super::method4();
            }
        }
    }
}
fn call_fn_outside() {
    top_mod1::method1();
    top_mod1::inner_mod1::method2();
}
```

top\_mod1外部的函數call\_fn\_outside()，有權訪問method1()，因為它是用pub修飾的。同樣也可以訪問method2()，因為inner\_mod1是pub的，而且method2也是pub的。而inner\_mod2不是pub的，所以外部的函數是沒法訪問method4的。但是call\_fn\_inside是有權訪問method4的，因為它在method4所處模組的子模組中。

### 重新導出

模組內的元素可以使用pub use重新匯出（re-export）。這也是Rust模組系統的一個重要特點。

```rust
mod top_mod1 {
    pub use self::inner_mod1::method1;
    mod inner_mod1 {
        pub use self::inner_mod2::method1;
        mod inner_mod2 {
            pub fn method1() {}
        }
    }
}
fn call_fn_outside() {
    top_mod1::method1();
}
```

在call\_fn\_outside函數中，我們調用了top\_mod1中的函數method1。可是我們注意到，method1其實不是在top\_mod1內部實現的，它只是把它內部inner\_mod1裡面的函數重新匯出了而已。

**pub use就是起這樣的作用，可以把元素當成模組的直接成員公開出去**。我們繼續往下看還可以發現，這個函數在inner\_mod1裡面也只是重新匯出的，它的真正實現是在inner\_mod2裡面。

這個機制可以讓我們輕鬆做到介面和實現的分離。我們可以先設計好一個模組的對外API，這個固定下來之後，它的具體實現是可以隨便改，不影響外部用戶的。我們可以把具體實現寫到任何一個子模組中，然後在當前模組重新匯出即可。對外部用戶來說，這沒什麼區別。

不過這個機制有個麻煩之處就是，如果具體實現嵌套在很深層次的子模組中的話，要把它匯出到最外面來，必須一層層地轉發，任何一層沒有重新匯出，都是無法達到目標的。

### pub限定可見性

Rust裡面用pub標記了的元素最終可能在哪一層可見，並不能很簡單地得出結論。因為它有可能被外面重新匯出。為了更清晰地限制可見性，Rust設計組又給pub關鍵字增加了下面的用法，可以明確地限定元素的可見性。

* method1用了pub（self）限制，那麼它最多只能被這個模組以及子模組使用，在模組外部調用或者重新匯出都會出錯。
* method2用了pub（super）限制，那麼它的可見性最多就只能到inner\_mod1這一層，在這層外面不能被調用或者重新匯出。
* 而method3用了pub（crate）限制，那麼它的可見性最多就只能到當前crate這一層，再繼續往外重新匯出，就會出錯。

```rust
mod top_mod {
    pub mod inner_mod1 {
        pub mod inner_mod2 {
            pub(self) fn method1() {}
            pub(super) fn method2() {}
            pub(crate) fn method3() {}
        }
        // Error:
        // pub use self::inner_mod2::method1;
        fn caller1() {
            // Error:
            // self::inner_mod2::method1();
        }
    }
    fn caller2() {
        // Error:
        // self::inner_mod1::inner_mod2::method2();
    }
}
// Error:
// pub use ::top_mod::inner_mod1::inner_mod2::method3;
```

## 同一資料夾內有main.rs與lib.rs

* <mark style="color:red;">使用crate::function呼叫定義在lib內的函數</mark>。
* 也可以用mod lib; lib::function方式呼叫。

```
| mod_example
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── lib.rs
│   ├── main.rs
│   └── worker.rs

```

```rust
// /mod_example/src/main.rs

// 方法一，使用crate_name::hello方式呼叫在lib.rs中的函數
mod worker;

fn main() {
    // mod的hello, 
    worker::hello();    //相對路徑
    crate::worker::hello(); //絕對路徑

    // 呼叫lib.rs內的hello
    mod_example::hello();//相對路徑
}

// 方法二，和一般mod相同，使用mod lib再呼叫
mod worker;
mod lib;

fn main() {
    // mod的hello
    worker::hello();
    // 呼叫lib.rs內的hello
    lib::hello();
}

// /mod_example/src/lib.rs
pub fn hello(){
    println!("hello lib");
}

// /mod_example/src/worker.rs
pub fn hello() {
    println!("worker file");
}
```

## 參考資料

* [https://tonydeng.github.io/2019/10/28/rust-mod/](https://tonydeng.github.io/2019/10/28/rust-mod/)
* [https://blog.csdn.net/wowotuo/article/details/107591501](https://blog.csdn.net/wowotuo/article/details/107591501)
