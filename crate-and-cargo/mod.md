# mod

## 簡介

mod（模組）是用於在crate內部繼續進行分層和封裝的機制。模組內部又可以包含模組。Rust中的模組是一個典型的樹形結構。每個crate會自動產生一個跟當前crate同名的模組，作為這個樹形結構的根節點。

在一個crate內部創建新模組的方式有下面幾種。

* 一個檔中創建內嵌模組。直接使用mod關鍵字即可，模組內容包含到大括弧內部：

```rust
mod name { fn items() {} … }
```

* 獨立的一個檔就是一個模組。檔案名即是模組名。
* 一個資料夾也可以創建一個模組。資料夾內部要有一個mod.rs文  件，這個檔就是這個模組的入口。

使用哪種方式編寫模組取決於當時的場景。

* 如果我們需要創建一個小型子模組，比如單元測試模組，那麼直接寫到一個檔內部就非常簡單而且直觀；
* 如果一個模組內容相對有點多，那麼把它單獨寫到一個檔內是更容易維護的；
* 如果一個模組的內容太多了，那麼把它放到一個資料夾中就更合理，因為我們可以把真正的內容繼續分散到更小的子模組中，而在mod.rs中直接重新匯出（re-export）。這樣mod.rs的源碼就大幅簡化，不影響外部的調用者。

## 範例

比如，我們有一個crate內部包含了兩個模組，一個是caller一個是worker。我們可以有幾種方案來實現。

### 方案一：直接把所有程式碼都寫到lib.rs裡面

```rust
// <lib.rs>
mod caller {
    fn call() {}
}
mod worker {
    fn work1() {}
    fn work2() {}
    fn work3() {}
}
```

### 方案二：把模組分到兩個不同的檔中

分別叫作caller.rs和worker.rs。那麼我們的項目就有了三個檔，它們的內容分別是：

```rust
// <lib.rs>
mod caller;
mod worker;
// <caller.rs>
fn call() {}
// <worker.rs>
fn work1() {}
fn work2() {}
fn work3() {}
```

<mark style="color:blue;">因為lib.rs是這個crate的入口，我們需要在這裡聲明它的所有子模組，否則caller.rs和worker.rs都不會被當成這個項目的程式碼編譯</mark>。

### 方案三：如果worker.rs這個檔包含的內容太多，我們還可以繼續分成幾個檔

```rust
// <lib.rs>
mod caller;
mod worker;
// <caller.rs>
fn call() {}
// <worker/mod.rs>
mod worker1;
mod worker2;
mod worker3;
// <worker/worker1.rs>
fn work1() {}
// <worker/worker2.rs>
fn work2() {}
// <worker/worker3.rs>
fn work3() {}
```

這樣就把一個模組繼續分成了幾個小模組。而且worker模組的拆分其實是不影響caller模組的，只要我們在worker模組中把它子模組內部的東西重新匯出（re-export）就可以了。這個是可見性控制的內容。

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

## use關鍵字

Rust裡面的路徑有不同寫法，它們代表的含義如下：

### 以::開頭的路徑，代表全域路徑。它是從crate的根部開始算。&#xD;

```rust
mod top_mod1 {
    pub fn f1() {}
}
mod top_mod2 {
    pub fn call() {
        // 當前crate下的top_mod1
        ::top_mod1::f1();
        // 也可以明確寫出crate
        crate::top_mod1::f1();
    }
}
```

### 以super關鍵字開頭的路徑是相對路徑。它是從上層模組開始算的

```rust
mod top_mod1 {
    pub fn f1() {}
    mod inner_mod1 {
        pub fn call() {
            // 當前模組 inner_mod1 的父級模組中的f1函數
            super::f1(); 
        }
    }
}
```

### 以self關鍵字開頭的路徑是相對路徑。它是從當前模組開始算的

```rust
mod top_mod1 {
    pub fn f1() {}
    pub fn call() {
        // 當前模組 top_mod1中的f1函數
        self::f1(); 
    }
}
```

## 簡化作用域

如果我們需要經常重複性地寫很長的路徑，那麼可以使用use語句把相應的元素引入到當前的作用域中來。

### 可以用大括弧，一句話引入多個元素

```rust
// 這句話引入了io / Read / Write 三個名字
use std::io::{self, Read, Write}; 
```

### use語句的大括弧可以嵌套使用

```rust
use a::b::{c, d, e::{f, g::{h, i}} };
```

### use語句可以使用星號，引入所有的元素

```rust
// 這句話引入了 std::io::prelude下面所有的名字
use std::io::prelude::*; 
```

### use語句不僅可以用在模組中，還可以用在函數、trait、impl等地方

```rust
fn call() {
    use std::collections::HashSet;
    let s = HashSet::<i32>::new();
}
```

### use語句允許使用as重命名，避免名字衝突

```rust
use std::result::Result as StdResult;
use std::io::Result as IoResult;
```
