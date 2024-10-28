# 路徑與IO

## 接收命令列參數

* [https://rustwiki.org/zh-CN/std/env/fn.args.html](https://rustwiki.org/zh-CN/std/env/fn.args.html)
* [https://rustwiki.org/zh-CN/std/env/struct.Args.html](https://rustwiki.org/zh-CN/std/env/struct.Args.html)

環境參數需要開發者通過`std::env` 模組取出。

以下結果中 Args 結構體中有一個 inner 陣列，只包含唯一的字串，代表了當前運行的程式所在的位置。

第一個元素傳統上是可執行檔案的路徑，但它可以設定為任意文字，甚至可能不存在。

如果處理程序的任何參數不是有效的 Unicode，則返回的迭代器將在迭代期間 panic。

```rust
fn main() {
    let args = std::env::args(); //  std::env::Args類型
    println!("{args:?}, len:{}", args.len());
    // 可走訪
    for arg in args {
        println!("{arg}");
    }
}
// 只使用cargo run時
// Args { inner: ["target/debug/mytest"] }, len: 1

// 使用cargo run -- one two 傳入兩個參數
// Args { inner: ["target/debug/mytest", "one", "two"] }, len:3
// target/debug/mytest
// one
// two
```

## 命令列輸入

* [https://rustwiki.org/zh-CN/std/io/index.html](https://rustwiki.org/zh-CN/std/io/index.html)
* [https://rustwiki.org/zh-CN/std/io/fn.stdin.html](https://rustwiki.org/zh-CN/std/io/fn.stdin.html)
* [https://rustwiki.org/zh-CN/std/io/struct.Stdin.html](https://rustwiki.org/zh-CN/std/io/struct.Stdin.html)

stdin()函數為當前處理程序的標準輸入建立一個新的控制代碼(handle)。返回的每個控制代碼都是對共享緩衝區的引用，該緩衝區的訪問通過互斥鎖進行同步。

`pub fn read_line(&self, buf: &mut String) -> Result<usize>` 鎖定此控制代碼並讀取輸入行，並將其新增到指定的緩衝區。Result的usize為讀取的位元組(byte)數。

```rust
use std::io::stdin; 

fn main() {
    let mut str_buf = String::new();
    // 使用者在此輸入文字，接下enter後結束輸入
    stdin()
        .read_line(&mut str_buf)
        .expect("Failed to read line.");
    println!("Your input line is \n{str_buf}");
}
```

## 檔案讀取

### 讀取文字檔

```rust
use std::fs;

fn main() {
    //讀取文字檔
    let text = fs::read_to_string("D:\\text.txt").unwrap();
    println!("{ text}");
}
```

### 讀取二進位檔

一次性讀取

```rust
use std::fs;

fn main() {
    let content = fs::read("D:\\text.txt").unwrap();
    println!("{content:?}");
}
```

對於一些底層程式來說，傳統的按流讀取的方式依然是無法被取代的，因為更多情況下檔案的大小可能遠超記憶體容量。

### 分段讀取(stream)

std::fs 模組中的 File 類是描述檔案的類，可以用於打開檔案。

打開檔案之後，我們可以使用 File 的 read 方法按流讀取檔案的下面一些位元組到緩衝區（緩衝區是一個 u8 陣列），讀取的位元組數等於緩衝區的長度。

std::fs::File 的 open 方法是"唯讀"打開檔案，並且沒有配套的 close 方法，因為 Rust 編譯器可以在檔案不再被使用時自動關閉檔案。

```rust
use std::fs;

fn main() {
    let mut buffer = [0u8; 5];
    let mut file = fs::File::open("D:\\text.txt").unwrap();
    file.read(&mut buffer).unwrap();
    println!("{buffer:?}");
    file.read(&mut buffer).unwrap();
    println!("{buffer:?}");
}
```

## 檔案寫入

檔案寫入分為一次性寫入和流式寫入。流式寫入需要打開檔案，打開方式有"新建"（create）和"追加"（append）兩種。

### 一次性寫入

一次性寫入請謹慎使用！因為它會直接刪除檔案內容（無論檔案多麼大）。如果檔案不存在就會建立檔案。

```rust
use std::fs;

fn main() {
    fs::write("D:\\text.txt", "FROM RUST PROGRAM")
        .unwrap();
}
```

等價寫法：

```rust
use std::fs::File;

fn main() {
    let mut file = File::create("D:\\text.txt").unwrap();
    file.write(b"FROM RUST PROGRAM").unwrap();
}
```

### append

File 類中不存在 append 靜態方法，但是我們可以使用 OpenOptions 來實現用特定方法打開檔案：

```rust
use std::fs::OpenOptions;

fn main() -> std::io::Result<()> {
   
    let mut file = OpenOptions::new()
            .append(true).open("D:\\text.txt")?;

    file.write(b" APPEND WORD")?;

    Ok(())
}
```

