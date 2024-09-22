# trait物件

## 簡介

trait物件在Rust中是指使用指標封裝了的 trait，比如`&SomeTrait` 和 `Box<SomeTrait>`。

因為`&SomeTrait`記憶體大小並不確定，因此需加上`dyn`關鍵字 。

```rust
trait Draw {
    fn draw(&self) -> String;
}

impl Draw for u8 {
    fn draw(&self) -> String {
        format!("u8: {}", *self)
    }
}

impl Draw for f64 {
    fn draw(&self) -> String {
        format!("f64: {}", *self)
    }
}

// 若 T 實現了 Draw 特徵， 則呼叫該函數時傳入的 Box<T> 可以被隱式轉換成函數參數簽名中的 Box<dyn Draw>
fn draw1(x: Box<dyn Draw>) {
    // 由於實現了 Deref 特徵，Box 智慧指針會自動解引用為它所包裹的值，然後呼叫該值對應的類型上定義的 `draw` 方法
    println!("draw1 {}", x.draw());
}

fn draw2(x: &dyn Draw) {
    println!("draw2 {}", x.draw());
}

fn main() {
    let x = 1.1f64;
    // do_something(&x);
    let y = 8u8;

    // x 和 y 的類型 T 都實現了 `Draw` 特徵，因為 Box<T> 可以在函數呼叫時隱式地被轉換為特徵對象 Box<dyn Draw> 
    // 基於 x 的值建立一個 Box<f64> 類型的智慧指針，指針指向的資料被放置在了堆上
    draw1(Box::new(x));
    // 基於 y 的值建立一個 Box<u8> 類型的智慧指針
    draw1(Box::new(y));
    draw2(&x);
    draw2(&y);
}
```



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
