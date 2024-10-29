# 自訂錯誤

可以透過實作 std::error::Error trait 來自訂錯誤型別。這樣做讓我們可以建立具有描述性且適合自己應用的錯誤訊息。此外，可以搭配 std::fmt::Display 和 std::fmt::Debug trait，方便將錯誤格式化並印出。

```rust
use std::fmt;
use std::error::Error;

// 自訂錯誤型別
#[derive(Debug)]
struct MyCustomError {
    details: String,
}

// 為 MyCustomError 實作 Error trait
impl Error for MyCustomError {}

// 為 MyCustomError 實作 Display trait，以便提供自訂的錯誤訊息
impl fmt::Display for MyCustomError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "錯誤發生：{}", self.details)
    }
}

// 提供建立錯誤的建構函式
impl MyCustomError {
    fn new(msg: &str) -> MyCustomError {
        MyCustomError { details: msg.to_string() }
    }
}

// 使用自訂錯誤的函式範例
fn might_fail(condition: bool) -> Result<(), MyCustomError> {
    if condition {
        Ok(())
    } else {
        Err(MyCustomError::new("發生了一些問題"))
    }
}

fn main() {
    match might_fail(false) {
        Ok(_) => println!("成功執行！"),
        Err(e) => println!("錯誤：{}", e),
    }
}

```

[https://theomn.com/rust-error-handling-for-pythonistas/](https://theomn.com/rust-error-handling-for-pythonistas/)

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
enum WhoUnfollowedError {
    /// Failed to complete an HTTP request.
    Http { source: reqwest::Error },
    /// Failed to read the cache file.
    DiskCacheRead { source: std::io::Error },
    /// Failed to update the cache file.
    DiskCacheWrite { source: std::io::Error },
}

impl fmt::Display for WhoUnfollowedError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            WhoUnfollowedError::Http { .. } => {
                write!(f, "Fetch failed")
            }
            WhoUnfollowedError::DiskCacheRead { .. } => {
                write!(f, "Disk cache read failed")
            }
            WhoUnfollowedError::DiskCacheWrite { .. } => {
                write!(f, "Disk cache write failed")
            }
        }
    }
}

impl Error for WhoUnfollowedError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            WhoUnfollowedError::Http { source } => Some(source),
            // The "disk" variants both have the same type for their source
            // fields so we can group them up in the same match arm.
            WhoUnfollowedError::DiskCacheRead { source }
            | WhoUnfollowedError::DiskCacheWrite { source } => Some(source),
        }
    }
}

impl From<reqwest::Error> for WhoUnfollowedError {
    fn from(other: reqwest::Error) -> WhoUnfollowedError {
        WhoUnfollowedError::Http { source: other }
    }
}
```

[https://edgl.dev/blog/wrapping-errors-in-rust/](https://edgl.dev/blog/wrapping-errors-in-rust/)
