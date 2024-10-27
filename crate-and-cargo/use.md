# use

## Mod 和 Use 的區別

關鍵字 use 用於將模組的內容匯入當前範圍。這意味著它將使模組中的所有函式都可以從此時開始呼叫。

mod 僅將另一個模組中的單個專案匯入當前範圍，因此可以根據需要呼叫或引用它，而不必擔心從現在開始可以訪問該模組中的任何其他內容。

<mark style="background-color:red;">它們之間的主要區別在於 use 從外部庫匯入模組，而 mod 建立只能在當前檔案中使用的內部模組</mark>。

### use 的特點

* 你可以使用 self 關鍵字將通用父模組和你想要使用的任何其他東西引入名稱空間。&#x20;
* 為避免身份問題，請使用 as 關鍵字進行更改。&#x20;
* 你可以使用類似 glob 的語法將多個物件帶入當前名稱空間：`use std::path::{self, Path, PathBuf};`

## use關鍵字

Rust裡面的路徑有不同寫法，它們代表的含義如下：

### 以::開頭的路徑，代表全域路徑。它是從crate的根部開始算。

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



## 參考資料
