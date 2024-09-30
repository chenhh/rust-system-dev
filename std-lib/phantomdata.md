---
description: std::marker::PhantomData
---

# PhantomData

Rust程式語言中所謂的幽靈資料(PhantomData)就是作為列舉或結構體中的資料欄位，<mark style="color:red;">純粹為了要給編譯器看所設計的資料型別</mark>。幽靈資料除了能處理上述的有用到泛型生命周期參數的特性外，也可以處理有用到泛型型別參數的特性。

例如現在有個特徵A，它有一個泛型生命周期參數'a，可以將符合這個生命周期參數的字串切片回傳。

```rust
trait A<'a> {
    fn as_str(&'a self) -> &'a str;
}
```

然後我們希望有個結構體B，可以儲存實作了這個特徵A的類別實體，因此我們會很直覺地如以下這樣定義結構體B：

```rust
// error
// 編譯器並不知道泛型生命周期參數'a到底是什麼，因為它在結構體B的資料欄位中並沒有被使用到
// T為實做了特徵A<'a>的類別
struct B<'a, T: A<'a>> {
    s: T
}
```

### 簡單解決方法：會造成使用記憶體變大

```rust
// 多儲存一個&'a str參考
struct B<'a, T: A<'a>> {
    s: T,
    _not_used: &'a str,
}

enum C<'a, T: A<'a>> {
    V1(T),
    _NotUsed(&'a str),
}
```

## 使用PhantomData

標準函式庫的marker模組有提供一個PhantomData結構體，就是專門用來解決這個問題的。PhantomData結構體有一個泛型型別參數T，這個T具體是什麼型別並沒有任何限制，且PhantomData結構體本身是不佔空間的(zero-sized)。

```rust
use std::marker::PhantomData;
 
struct B<'a, T: A<'a>> {
    s: T,
    _not_used: PhantomData<&'a str>,
}
 
enum C<'a, T: A<'a>> {
    V1(T),
    _NotUsed(PhantomData<&'a str>),
}
```

不用擔心具體要給PhantomData的資料欄位什麼值，通通傳入PhantomData就好了。

```rust
impl<'a, T: A<'a>> B<'a, T> {
    // 建立結構體B的實體時，在PhantomData的資料欄位中，直接指派了PhantomData給它
    fn new(s: T) -> Self {
        B {
            s,
            _not_used: PhantomData,
        }
    }
}

// 如果是列舉的話，我們甚至可以不必管PhantomData的資料欄位
impl<'a, T: A<'a>> C<'a, T> {
    fn v1(s: T) -> Self {
        C::V1(s)
    }
}
```

## 參考資料

* [https://doc.rust-lang.org/std/marker/struct.PhantomData.html](https://doc.rust-lang.org/std/marker/struct.PhantomData.html)
* [https://magiclen.org/rust-phantomdata/](https://magiclen.org/rust-phantomdata/)
