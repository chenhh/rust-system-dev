# 路徑與IO

## 路徑

Path 結構體代表了底層檔案系統的檔案路徑。Path 分為兩種：

* `posix::Path`，針對 類 UNIX 系統；
* 以及 `windows::Path`，針對 Windows。

prelude 會選擇並輸出符合平台類型 的 `Path` 種類。

`Path` 可從 `OsStr` 類型創建，並且它提供數種方法，用於獲取路徑指向的檔案/目錄 的資訊。

注意 Path 在內部並不是用 UTF-8 字串表示的，而是存儲為若干位元組（Vec）的 vector。因此，將 Path 轉化成 \&str 並非零開銷的（free），且可能失敗（因此它 返回一個 Option）。

```rust
use std::path::Path;

fn main() {
    // 從 `&'static str` 創建一個 `Path`
    let path = Path::new(".");

    // `display` 方法返回一個可顯示（showable）的結構體
    let display = path.display();
    println!("display: {:?}", display); // "."

    // `join` 使用操作系統特定的分隔符來合並路徑到一個字節容器，並返回新的路徑
    let new_path = path.join("a").join("b");

    // 將路徑轉換成一個字串切片
    match new_path.to_str() {
        None => panic!("new path is not a valid UTF-8 sequence"),
        Some(s) => println!("new path is {}", s),   // ./a/b
    }
}
```
