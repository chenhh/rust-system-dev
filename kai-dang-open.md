# 開檔(open)

## File結構體

`File` 結構體表示一個被打開的檔案（它包裹了一個檔案描述符），並賦予了對所表示的檔案的讀寫能力。

由於在進行檔案 I/O（輸入/輸出）操作時可能出現各種錯誤，因此 `File` 的所有方法都 返回 `io::Result` 類型，它是 `Result<T, io::Error>` 的別名。

這使得所有 I/O 操作的失敗都變成顯式的。借助這點，程式員可以看到所有的失敗路徑，並被鼓勵主動地處理這些情形。

## Open

open 靜態方法能夠以唯讀模式（read-only mode）打開一個檔案。File 擁有資源，即檔案描述符（file descriptor），它會在自身被 drop 時關閉檔案。

```rust
use std::fs::File;
use std::io::prelude::*;
use std::path::Path;

fn main() {
    // 創建指向所需的檔案的路徑
    let path = Path::new("hello.txt");
    let display = path.display();

    // 以唯讀方式打開路徑，返回 `io::Result<File>`
    let mut file = match File::open(&path) {
        // `io::Error` 的 `description` 方法返回一個描述錯誤的字串。
        Err(why) => panic!("無法開啟 {}: {}", display, why.to_string()),
        Ok(file) => file,
    };

    // 讀取檔案內容到一個字串，返回 `io::Result<usize>`
    let mut s = String::new();
    match file.read_to_string(&mut s) {
        Err(why) => panic!("couldn't read {}: {}", display, why.to_string()),
        Ok(_) => print!("{} contains:\n{}", display, s),
    }
    // `file` 離開作用域，並且 `hello.txt` 檔案將被關閉。
}
```

## create

create 靜態方法以只寫模式（write-only mode）打開一個檔案。若檔案已經存在，則舊內容將被銷毀。否則，將創建一個新檔案。

還有一個更通用的 open\_mode 方法，這能夠以其他方式來來打開 檔案，如：read+write（讀+寫），追加（append），等等。

```rust
static LOREM_IPSUM: &'static str =
    "Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod
tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam,
quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo
consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse
cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non
proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
";

use std::fs::File;
use std::io::prelude::*;
use std::path::Path;

fn main() {
    let path = Path::new("out/lorem_ipsum.txt");
    let display = path.display();

    // 以只寫模式開啟檔案，返回 `io::Result<File>`
    let mut file = match File::create(&path) {
        Err(why) => panic!("couldn't create {}: {}", display, why.to_string()),
        Ok(file) => file,
    };

    // 將 `LOREM_IPSUM` 字串寫進 `file`，返回 `io::Result<()>`
    match file.write_all(LOREM_IPSUM.as_bytes()) {
        Err(why) => {
            panic!("couldn't write to {}: {}", display, why.to_string())
        }
        Ok(_) => println!("successfully wrote to {}", display),
    }
}
```

## lines以行回傳迭代器

方法 lines() 在檔案的行上返回一個迭代器。File::open 需要一個泛型 AsRef。這正是 read\_lines() 期望的輸入。

```rust
use std::fs::File;
use std::io::{self, BufRead};
use std::path::Path;

fn main() {
    // 在生成輸出之前，檔案主機必須存在於當前路徑中
    if let Ok(lines) = read_lines("./hosts") {
        // 使用迭代器，返回一個（可選）字串
        for line in lines {
            if let Ok(ip) = line {
                println!("{}", ip);
            }
        }
    }
}

// 輸出包裹在 Result 中以允許匹配錯誤，
// 將迭代器返回給檔案行的讀取器（Reader）。
fn read_lines<P>(filename: P) -> io::Result<io::Lines<io::BufReader<File>>>
where
    P: AsRef<Path>,
{
    let file = File::open(filename)?;
    Ok(io::BufReader::new(file).lines())
}
```
