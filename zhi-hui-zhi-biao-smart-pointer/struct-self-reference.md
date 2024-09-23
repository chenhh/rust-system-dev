# 結構體自引用

## 範例：結構體自引用

```rust
struct SelfRef<'a> {
    value: String,

    // 該引用指向上面的value
    pointer_to_value: &'a str,
}

fn main(){
    let s = "aaa".to_string();
    // 因為我們試圖同時使用值和值的引用，最終所有權轉移和借用一起發生了
    let v = SelfRef {
        value: s,
        pointer_to_value: &s //error
    };
}
```

如果兩個放在一起會報錯，那就分開它們。對，終極大法就這麼簡單，當然思路上的簡單不代表實現上的簡單，最終結果就是導致程式碼複雜度的上升。

## 解法1：使用Option

使用 Option 分兩步來實現。

Option 這個方法可以工作，但是這個方法的限制較多，例如從一個函數建立並返回它是不可能的。

```rust
#[derive(Debug)]
struct WhatAboutThis<'a> {
    name: String,
    nickname: Option<&'a str>,
}

fn main() {
    let mut tricky = WhatAboutThis {
        name: "Annabelle".to_string(),
        nickname: None,
    };
    tricky.nickname = Some(&tricky.name[..4]);

    println!("{:?}", tricky);
}
```

### 解法2：unsafe 實現

在 pointer\_to\_value 中直接儲存裸指針，而不是 Rust 的引用，因此不再受到 Rust 借用規則和生命週期的限制，而且實現起來非常清晰、簡潔。但是缺點就是，通過指針獲取值時需要使用 unsafe 程式碼。

```rust
#[derive(Debug)]
struct SelfRef {
    value: String,
    pointer_to_value: *const String,
}

impl SelfRef {
    fn new(txt: &str) -> Self {
        SelfRef {
            value: String::from(txt),
            pointer_to_value: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.value;
        self.pointer_to_value = self_ref;
    }

    fn value(&self) -> &str {
        &self.value
    }

    fn pointer_to_value(&self) -> &String {
        assert!(!self.pointer_to_value.is_null(),
            "Test::b called without Test::init being called first");
        unsafe { &*(self.pointer_to_value) }
    }
}

fn main() {
    let mut t = SelfRef::new("hello");
    t.init();
    // 列印值和指針位址
    println!("{}, {:p}", t.value(), t.pointer_to_value());
}
```

## 解法3: 無法被移動的 Pin

pin可以固定住一個值，防止該值在記憶體中被移動。

自引用最麻煩的就是建立引用的同時，值的所有權會被轉移，而通過 Pin 就可以很好的防止這一點：

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::ptr::NonNull;

// 下面是一個自引用資料結構體，因為 slice 欄位是一個指針，指向了 data 欄位
// 我們無法使用普通引用來實現，因為違背了 Rust 的編譯規則
// 因此，這裡我們使用了一個裸指針，通過 NonNull 來確保它不會為 null
struct Unmovable {
    data: String,
    slice: NonNull<String>,
    _pin: PhantomPinned,
}

impl Unmovable {
    // 為了確保函數返回時資料的所有權不會被轉移，我們將它放在堆上，唯一的訪問方式就是通過指針
    fn new(data: String) -> Pin<Box<Self>> {
        let res = Unmovable {
            data,
            // 只有在資料到位時，才建立指針，否則資料會在開始之前就被轉移所有權
            slice: NonNull::dangling(),
            _pin: PhantomPinned,
        };
        let mut boxed = Box::pin(res);

        let slice = NonNull::from(&boxed.data);
        // 這裡其實安全的，因為修改一個欄位不會轉移整個結構體的所有權
        unsafe {
            let mut_ref: Pin<&mut Self> = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).slice = slice;
        }
        boxed
    }
}

fn main() {
    let unmoved = Unmovable::new("hello".to_string());
    // 只要結構體沒有被轉移，那指針就應該指向正確的位置，而且我們可以隨意移動指針
    let mut still_unmoved = unmoved;
    assert_eq!(still_unmoved.slice, NonNull::from(&still_unmoved.data));

    // 因為我們的類型沒有實現 `Unpin` 特徵，下面這段程式碼將無法編譯
    // let mut new_unmoved = Unmovable::new("world".to_string());
    // std::mem::swap(&mut *still_unmoved, &mut *new_unmoved);
}
```
