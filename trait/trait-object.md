# trait物件

## 簡介

trait物件在Rust中是指使用指標封裝了的 trait，比如`&SomeTrait` 和 `Box<SomeTrait>`。

因為`&SomeTrait`記憶體大小並不確定，因此需加上`dyn`關鍵字 。

```rust
trait Foo {
    fn method(&self) -> String;
}
// 兩個類型都實現相同的trait
impl Foo for u8 {
    fn method(&self) -> String {
        format!("u8: {}", *self)
    }
}
impl Foo for String {
    fn method(&self) -> String {
        format!("string: {}", *self)
    }
}

// 參數為有實現Foo trait的類型即可
fn do_something(x: &dyn Foo) -> String {
    x.method()
}
fn main() {
    let x = "Hello".to_string();
    println!("{}", do_something(&x));
    let y = 8u8;
    println!("{}", do_something(&y));
}
```

## 物件安全(object safe)

並不是所有的trait都能作為trait物件使用。

如果一個trait方法是object safe的，它需要滿足：方法有Self: Sized約束， 或者&#x20;

同時滿足以下所有條件：&#x20;

* 沒有泛型參數
* 不是靜態函數
* 除了self之外的其它參數和返回值不能使用Self類型

如果一個trait是object-safe的，它需要滿足：

* 所有的方法都是object-safe的，
* 並且 trait 不要求 Self: Sized 約束。
