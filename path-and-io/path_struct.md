# 路徑(path)

## 路徑

* [https://doc.rust-lang.org/std/path/struct.Path.html](https://doc.rust-lang.org/std/path/struct.Path.html)
* [https://rustwiki.org/zh-CN/std/path/struct.Path.html](https://rustwiki.org/zh-CN/std/path/struct.Path.html)
* [https://doc.rust-lang.org/std/path/struct.PathBuf.html](https://doc.rust-lang.org/std/path/struct.PathBuf.html)
* [https://rustwiki.org/zh-CN/std/path/struct.PathBuf.html](https://rustwiki.org/zh-CN/std/path/struct.PathBuf.html)

Path 結構體代表了底層檔案系統的檔案路徑。Path 分為兩種：

* `posix::Path`，針對 類 UNIX 系統；
* 以及 `windows::Path`，針對 Windows。

prelude 會選擇並輸出符合平台類型 的 `Path` 種類。

`Path` 可從 `OsStr` 類型創建，並且它提供數種方法，用於獲取路徑指向的檔案/目錄 的資訊。

* Path 路徑的切片 (類似於 str)。
* PathBuf 擁有的可變路徑 (類似於 String)。

PathBuf 是 Rust 標準庫中 `std::path` 模組提供的一個類型，用於表示檔案路徑的可變版本。它是 Path 的所有者形式，提供了許多方便的方法來操作和處理檔案路徑。

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

要構建或修改路徑，請使用 PathBuf：

```rust
use std::path::PathBuf;

// 這種方式有效...
let mut path = PathBuf::from("c:\\");

path.push("windows");
path.push("system32");

path.set_extension("dll");

// ... 但是如果您不瞭解所有內容，則最好使用push。
// 如果您這樣做，則這種方法更好：
let path: PathBuf = ["c:\\", "windows", "system32.dll"].iter().collect();
```

## Path常用方法

```rust
use std::path::PathBuf;

/* 建立 PathBuf */

// 使用 new 方法建立一個空的 PathBuf
let mut path = PathBuf::new();

// 從字串字面量建立 PathBuf
let path = PathBuf::from("/some/path/to/file");

// 從作業系統特定的字串建立 PathBuf
let os_str = std::ffi::OsStr::new("/some/path/to/file");
let path = PathBuf::from(os_str);

/* 追加和拼接路徑 */

// 使用 push 方法追加路徑
let mut path = PathBuf::from("/some/path");
path.push("to");
path.push("file");
assert_eq!(path, PathBuf::from("/some/path/to/file"));

// 使用 join 方法拼接路徑
let path = PathBuf::from("/some/path").join("to").join("file");
assert_eq!(path, PathBuf::from("/some/path/to/file"));

// 用iterator直接合併路徑
let path: PathBuf = ["c:\\", "windows", "system32.dll"].iter().collect();

/* 獲取路徑的各個部分 */
// 獲取檔名
let path = PathBuf::from("/some/path/to/file.txt");
assert_eq!(path.file_name(), Some(std::ffi::OsStr::new("file.txt")));

// 獲取父路徑
let parent = path.parent();
assert_eq!(parent, Some(Path::new("/some/path/to")));

// 獲取擴展名
assert_eq!(path.extension(), Some(std::ffi::OsStr::new("txt")));

// 獲取路徑的元件
for component in path.components() {
    println!("{:?}", component);
}

// 獲取路徑的根元件
assert_eq!(path.root(), Some("/"));

/* 轉換為其他類型 */
use std::ffi::OsStr;

// 轉換為字串切片
let path = PathBuf::from("/some/path/to/file.txt");
let path_str = path.to_str().unwrap(); // 返回 Option<&str>
assert_eq!(path_str, "/some/path/to/file.txt");

// 轉換為 OsStr
let os_str = path.as_os_str();
assert_eq!(os_str, OsStr::new("/some/path/to/file.txt"));

/* 標準路徑操作 */
// 獲取絕對路徑
let path = PathBuf::from("file.txt");
let absolute_path = path.canonicalize().unwrap(); // 返回 Result<PathBuf>
println!("{:?}", absolute_path);

// 判斷路徑是否存在
let exists = path.exists();
println!("Path exists: {}", exists);

// 判斷是否是檔案
let is_file = path.is_file();
println!("Is file: {}", is_file);

// 判斷是否是目錄
let is_dir = path.is_dir();
println!("Is directory: {}", is_dir);
```

