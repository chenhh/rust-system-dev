# Cow(copy on write)



## Cow (copy on write)

Clone On Write 的智慧指標，修改指標指向的資料時，才會複製一份資料。

* Cow 是一個 enum，會是 borrowed 或 owned data 其中一種。&#x20;
* Cow 的 T 需實作 `ToOwned trait`，可視為需要能被 clone。

```rust
pub enum Cow<'a, B: ?Sized + 'a>
    where B: ToOwned
{
    /// Borrowed data.
    Borrowed &'a B), 
    /// Owned data.
    Owned(<B as ToOwned>::Owned),
}
```

事實上，除了 trait實現外，Cow 只有 `into_owned` 與 `to_mut` 兩個方法。

```rust
use std::borrow::Cow;
fn main() {
    let s = "Hello world!";
    let cow = Cow::Borrowed(s); // 引用 s
    assert_eq!(
        cow.into_owned(), // 透過 `into_owned()` 轉換成 owned data
        String::from(s)
    );
    let mut cow = Cow::Borrowed("foo");
    cow.to_mut().make_ascii_uppercase(); // 利用 to_mut() 複製一份資料
    assert_eq!(
        cow,
        Cow::Owned(String::from("FOO")) as Cow<str> // 已與原始資料不同
    );
}
```

### 什麼時候該用 Cow

* 你遇到 clone 的效能瓶頸，希望減少 clone 的次數。
* 你不想要處理多重指標的問題，但也不想要全部都 clone。
* 你想要避免 Mutex 互斥鎖效能不彰的問題。
