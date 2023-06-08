# 測試

## 簡介

Rust 中的測試函數是用來驗證非測試代碼是否按照期望的方式運行的。測試函數體通常執行如下三種操作：

* 設置任何所需的數據或狀態。
* 運行需要測試的代碼。
* 斷言(assert)其結果是我們所期望的。

測試有三種風格：

* 單元測試。
* 文檔測試。
* 整合測試。

有時僅在測試中才需要一些依賴（比如基準測試相關的）。這種依賴要寫在 Cargo.toml 的 \[dev-dependencies] 部分。這些依賴不會傳播給其他依賴於這個包的包。

## 如何撰寫測試？

加上`#[cfg(test)]`的模組在執行`cargo test`指令的時候其底下有被加上`#[test]`的函數會被執行。

Rust的「測試」就是一個函數，這個函數被用來驗證其它非測試函數的函數(沒有加上#\[test]的函數)所實作的功能。

測試函數的主體，通常會有以下三個行為：

1. 建構測試時需要的資料和狀態。
2. 執行我們要測試的程式(例如要被測試的函數)。
3. 驗證程式執行之後的結果是不是跟我們預期的一樣。

## 單元測試

大多數單元測試都會被放到一個叫 `tests` 的、帶有 `#[cfg(test)]` 屬性的模塊(mod)中，測試函數要加上 `#[test]` 屬性。完成後，可以使用 cargo test 來運行測試。

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// 這個加法函式寫得很差，本例中我們會使它失敗。
#[allow(dead_code)]
fn bad_add(a: i32, b: i32) -> i32 {
    a - b
}

#[cfg(test)]
mod tests {
    // 注意這個慣用法：在 tests 模組中，從外部作用域匯入所有名字。
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(1, 2), 3);
    }

    #[test]
    fn test_bad_add() {
        // 這個斷言會導致測試失敗。注意私有的函式也可以被測試！
        assert_eq!(bad_add(1, 2), 3);
    }
}
```

### 測試 panic

一些函數應當在特定條件下 panic。為測試這種行為，可使用 `#[should_panic]` 屬性。這 個屬性接受可選參數 expected = 以指定 `panic` 時的訊息。如果你的函數能以多種方式 `panic`，這個屬性就保證了你在測試的確實是所指定的 `panic`。

```rust
pub fn divide_non_zero_result(a: u32, b: u32) -> u32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    } else if a < b {
        panic!("Divide result is zero");
    }
    a / b
}

#[cfg(test)]
mod tests {
    // 注意這個慣用法：在 tests 模組中，從外部作用域匯入所有名字。
    use super::*;

     #[test]
    fn test_divide() {
        assert_eq!(divide_non_zero_result(10, 2), 5);
    }

    #[test]
    #[should_panic]
    fn test_any_panic() {
        divide_non_zero_result(1, 0);
    }

    #[test]
    #[should_panic(expected = "Divide result is zero")]
    fn test_specific_panic() {
        divide_non_zero_result(1, 10);
    }
}
```

### 執行特定的測試

要執行特定的測試，只要把測試名稱傳給 `cargo test` 命令就可以了。如上範例 `cargo test test_any_panic`。

要運行多個測試，可以僅指定測試名稱中的一部分，用它來匹配所有要運行的測試。如`cargo test panic`。

### 忽略測試

可以把屬性 `#[ignore]` 賦予測試以排除某些測試，或者使用 `cargo test -- --ignored` 命令來運行它們。

```rust
 #[test]
    #[ignore]
    fn ignored_test() {
        assert_eq!(add(0, 0), 0);
    }
```

## 測試的組織結構

Rust 傾向於根據測試的兩個主要分類來考慮問題：單元測試（unit tests）與 整合測試（integration tests）。

* 單元測試傾向於更小而更集中，在隔離的環境中一次測試一個模塊，或者是測試私有介面。
* 而整合測試對於你的庫來說則完全是外部的。它們與其他外部代碼一樣，通過相同的方式使用你的代碼，只測試公有介面而且每個測試都有可能會測試多個模塊。

## 文檔測試

為 Rust 工程編寫文檔的主要方式是在程式碼中寫注釋。文檔注釋使用 markdown 語 書寫，支援程式塊。Rust 很注重正確性，這些注釋中的程式碼塊也會被編譯並且用作測試。

````rust
/// 第一行是對函式的簡短描述。
///
/// 接下來數行是詳細文件。程式碼塊用三個反引號開啟，Rust 會隱式地在其中新增
/// `fn main()` 和 `extern crate <cratename>`。比如測試 `doccomments` crate：
///
/// ```
/// let result = doccomments::add(2, 3);
/// assert_eq!(result, 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

/// 文件註釋通常可能帶有 "Examples"、"Panics" 和 "Failures" 這些部分。
///
/// 下面的函式將兩數相除。
///
/// # Examples
///
/// ```
/// let result = doccomments::div(10, 2);
/// assert_eq!(result, 5);
/// ```
///
/// # Panics
///
/// 如果第二個引數是 0，函式將會 panic。
///
/// ```rust,should_panic
/// // panics on division by zero
/// doccomments::div(10, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    }

    a / b
}
````

## 整合測試

函數級和模組級的測試，程式碼是與要測試的模組（編譯單元）寫在相同的檔案中，一般做的是白盒測試。

整合測試是 crate 外部的測試，並且僅使用 crate 的公共介面，就像其他使用 該 crate 的程式那樣。整合測試的目的是檢驗你的庫的各部分是否能夠正確地協同工作。

cargo 在與 src 同級別的 tests 目錄尋找整合測試。

```
// 檔案 src/lib.rs：
// 在一個叫做 'adder' 的 crate 中定義此函式。
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// 包含測試的檔案：tests/integration_test.rs：
#[test]
fn test_add() {
    assert_eq!(adder::add(3, 2), 5);
}
```

tests 目錄中的每一個 Rust 原始檔都被編譯成一個單獨的 crate。在整合測試中要想 共享代碼，一種方式是創建具有公用函數的模塊，然後在測試中導入並使用它。

帶有共用代碼的模塊遵循和普通的[模塊](https://rustwiki.org/zh-CN/rust-by-example/mod.html)一樣的規則，所以完全可以把公共模塊 寫在 `tests/common/mod.rs` 檔案中。

```rust
檔案 tests/common.rs:
pub fn setup() {
    // 一些配置程式碼，比如建立檔案/目錄，開啟伺服器等等。
}

// 包含測試的檔案：tests/integration_test.rs
// 匯入共用模組。
mod common;

#[test]
fn test_add() {
    // 使用共用模組。
    common::setup();
    assert_eq!(adder::add(3, 2), 5);
}
```
