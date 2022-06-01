# Option與Result互動

## 處理多種錯誤類型

有時 `Option` 需要和 `Result` 進行互動，或是 `Result<T, Error1>` 需要和 `Result<T, Error2>` 進行互動。在這類情況下，我們想要以一種方式來管理不同的錯誤 類型，使得它們可組合且易於互動。

```rust
fn double_first(vec: Vec<&str>) -> i32 {
    let first = vec.first().unwrap(); //  返回一個 Option
    2 * first.parse::<i32>().unwrap() //  返回一個 Result<i32, ParseIntError>
}

fn main() {
    let numbers = vec!["42", "93", "18"];
    let empty: Vec<i32> = vec![];
    let strings = vec!["tofu", "93", "18"];
    
    println!("The first doubled is {}", double_first(numbers));
    
    println!("The first doubled is {}", double_first(empty));
    // 錯誤1：輸入 vector 為空
    
    println!("The first doubled is {}", double_first(strings));
    // 錯誤2：此元素不能解析成數字
}
```

有時候我們不想再處理錯誤（比如使用 ? 的時候），但如果 Option 是 None 則繼續處理錯誤。一些組合運算元可以讓我們輕松地交換 Result 和 Option。

```rust
use std::num::ParseIntError;

fn double_first(vec: Vec<&str>) -> Result<Option<i32>, ParseIntError> {
    let opt = vec.first().map(|first| {
        first.parse::<i32>().map(|n| 2 * n)
    });

    opt.map_or(Ok(None), |r| r.map(Some))
}
```

## 從 Option 中取出 Result

處理混合錯誤類型的最基本的手段就是讓它們互相包含。

```rust
fn double_first(vec: Vec<&str>) -> Option<Result<i32, ParseIntError>> {
    vec.first().map(|first| {
        first.parse::<i32>().map(|n| 2 * n)
    })
}
```

## 返回值由Option 轉 Result

`ok_or()` 可以把 Option 類型轉換為 Result。

```rust
fn foo() -> Option<i32> {
    None
}

fn bar() -> Result<i32, String> {
    foo().ok_or("error".to_string())?;
    Ok(689)
}

fn main() {
    println!("{:?}", bar()); //Err("error")
}
```

## 返回值由Result轉Option

如果想提前丟棄計算結果，可以用 `.ok()?` 或`.err()?`方法。

```rust
pub fn ok(self) -> Option<T> {
        match self {
            Ok(x) => Some(x),
            Err(_) => None,
        }
    }
    
pub fn err(self) -> Option<E> {
        match self {
            Ok(_) => None,
            Err(x) => Some(x),
        }
    }
```

```rust
fn foo() -> Result<i32, i32> {
    Err(123)
}

fn bar() -> Option<i32> {
    foo().ok()?; // Ok時取值，否則傳回None
    Some(689)
}

fn main() {
    println!("{:?}", bar()); //None
}
```
