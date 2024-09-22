# 格式化輸出

## 格式化輸出

列印操作由 std::fmt 裡面所定義的一系列巨集來處理，包括：

* `format!`：將格式化文字寫到字串（String）。（譯註：字串是返回值不是參數。）
* `print!`：與 format! 類似，但將文字輸出到控制台（io::stdout）。
* `println!`: 與 print! 類似，但輸出結果追加一個換行符。
* `eprint!`：與 format! 類似，但將文字輸出到標准錯誤（io::stderr）。
* `eprintln!`：與 eprint! 類似，但輸出結果追加一個換行符。

這些巨集都以相同的做法解析（parse）文字。另外有個優點是格式化的正確性會在編譯時檢查。

```rust
fn main() {
    // 通常情況下，`{}` 會被任意變數內容所替換。
    // 變數內容會轉化成字串。
    println!("{} days", 31);
    // 不加尾碼的話，31 就自動成為 i32 類型。
    // 你可以添加尾碼來改變 31 的類型（例如使用 31i64 聲明 31 為 i64 類型）。
    // 用變數替換字串有多種寫法。
    // 比如可以使用位置參數。
    println!("{0}, this is {1}. {1}, this is {0}", "Alice", "Bob");
    // 可以使用具名引數。
    println!(
        "{subject} {verb} {object}",
        object = "the lazy dog",
        subject = "the quick brown fox",
        verb = "jumps over"
    );
    // 可以在 `:` 後面指定特殊的格式。
    println!("{} of {:b} people know binary, the other half don't", 1, 2);
    // 你可以按指定寬度來右對齊文本。
    // 下面語句輸出 " 1"，5 個空格後面連著 1。
    println!("{number:>width$}", number = 1, width = 6);
    // 你可以在數位左邊補 0。下面語句輸出 "000001"。
    println!("{number:>0width$}", number = 1, width = 6);
    // println! 會檢查使用到的參數數量是否正確。
    println!("My name is {0}, {1} {0}", "Bond");
    // 改正 ^ 補上漏掉的參數："James"
    // 創建一個包含單個 `i32` 的結構體（structure）。命名為 `Structure`。
    #[allow(dead_code)]
    struct Structure(i32);
    // 但是像結構體這樣的自訂類型需要更複雜的方式來處理。
    // 下麵語句無法執行。
    println!("This struct `{}` won't print...", Structure(3));
    // 改正 ^ 注釋掉此行。
}

```

## 自訂類型進行自訂的格式化輸出

在開發過程中，往往只要使用 #\[derive(Debug)] 對我們的自訂類型進行標註，即可實現列印輸出的功能。

```rust
#[derive(Debug)]
struct Point{
    x: i32,
    y: i32
}
fn main() {
    let p = Point{x:3,y:3};
    println!("{:?}",p);
}
```

為了使用者更好的閱讀理解我們的類型，此時就要為自訂類型實現 `std::fmt::Display` 特徵：

```rust
#![allow(dead_code)]

use std::fmt;
use std::fmt::{Display};

#[derive(Debug,PartialEq)]
enum FileState {
  Open,
  Closed,
}

#[derive(Debug)]
struct File {
  name: String,
  data: Vec<u8>,
  state: FileState,
}

impl Display for FileState {
   fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
     match *self {
         FileState::Open => write!(f, "OPEN"),
         FileState::Closed => write!(f, "CLOSED"),
     }
   }
}

impl Display for File {
   fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
      write!(f, "<{} ({})>",
             self.name, self.state)
   }
}

impl File {
  fn new(name: &str) -> File {
    File {
        name: String::from(name),
        data: Vec::new(),
        state: FileState::Closed,
    }
  }
}

fn main() {
  let f6 = File::new("f6.txt");
  //...
  println!("{:?}", f6);
  println!("{}", f6);
}
```
