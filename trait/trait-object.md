# trait物件

## 簡介

trait物件在Rust中是指使用指標封裝了的特徵，比如`&SomeTrait` 和 `Box<SomeTrait>`。可以指向實現了該特徵的實體。

注意 `dyn` 不能單獨作為特徵物件定義，例如下面的程式碼編譯器會報錯，原因是特徵物件可以是任意實現了某個特徵的類型，編譯器在編譯期不知道該類型的大小，不同的類型大小是不同的。而`&dyn` 和 `Box<dyn>` 在編譯期都是已知大小，所以可以用作特徵物件的定義。

```rust
//error, 必須修改為x: &dyn Draw或x: Box<dyn Draw>
fn draw2(x: dyn Draw) {
    x.draw();
}
```

只要看到dyn關鍵字就知道是Trait而非Struct(?)。

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
/*
draw1 f64: 1.1
draw1 u8: 8
draw2 f64: 1.1
draw2 u8: 8
*/
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

## 應用：利用特徵物件儲存相異結構的vec

```rust
// UI物件
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // 繪製按鈕的程式碼
    }
}
// UI物件
struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // 繪製SelectBox的程式碼
    }
}

// Button與SelectBox都實現了Draw特徵，
// 使用特徵物件存在components向量中。
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

// UI 元件渲染在螢幕上
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}

// 如果以泛型實作時，componets只能存單一種類的UI物件
pub struct Screen<T: Draw> {
    pub components: Vec<T>,
}

impl<T> Screen<T>
    where T: Draw {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

## 特徵物件的動態分發

泛型是在編譯期完成處理的：編譯器會為每一個泛型參數對應的具體類型生成一份程式碼，這種方式是**靜態分發(static dispatch)**，因為是在編譯期完成的，對於運行期效能完全沒有任何影響。

與靜態分發相對應的是動態分發(dynamic dispatch)，在這種情況下，直到執行階段，才能確定需要呼叫什麼方法。之前程式碼中的關鍵字 <mark style="color:red;">dyn 正是在強調這一“動態”的特點</mark>。

<figure><img src="../.gitbook/assets/image.png" alt="" width="375"><figcaption><p>靜態/動態分發</p></figcaption></figure>

如 \&dyn Draw、Box 雖然特徵物件沒有固定大小，但它的引用類型的大小是固定的，它由兩個指標組成（ptr 和 vptr），因此佔用兩個指標大小。

一個指標 ptr 指向實現了特徵 Draw 的具體類型的實例，也就是當作特徵 Draw 來用的類型的實例，比如類型 Button 的實例、類型 SelectBox 的實例。

另一個指標 vptr 指向一個虛表 vtable，vtable 中儲存了類型 Button 或類型 SelectBox 的實例。對於可以呼叫的實現於特徵 Draw 的方法。當呼叫方法時，直接從 vtable 中找到方法並呼叫。之所以要使用一個 vtable 來儲存各實例的方法，是因為實現了特徵 Draw 的類型有多種，這些類型擁有的方法各不相同，當將這些類型的實例都當作特徵 Draw 來使用時(此時，它們全都看作是特徵 Draw 類型的實例)，有必要區分這些實例各自有哪些方法可呼叫。

## 物件安全(object safe)

並不是所有的特徵都能作為trait物件使用。

如果一個特徵是物件安全(object safe)的，它需要滿足：方法有Self: Sized約束， 或者&#x20;

同時滿足以下所有條件：&#x20;

* 沒有泛型參數。
* 除了self之外的其它參數和返回值不能使用Self類型。

<mark style="color:red;">物件安全對於特徵物件是必須的，因為一旦有了特徵物件，就不再需要知道實現該特徵的具體類型是什麼了</mark>。

如果特徵方法返回了具體的 Self 類型，但是特徵對象忘記了其真正的類型，那這個 Self 就非常尷尬，因為沒人知道它是誰了。

但是對於泛型類型參數來說，當使用特徵時其會放入具體的類型參數：此具體類型變成了實現該特徵的類型的一部分。而當使用特徵對象時其具體類型被抹去了，故而無從得知放入泛型參數類型到底是什麼。

如果一個trait是object-safe的，它需要滿足：

* 所有的方法都是object-safe的，
* 並且 trait 不要求 Self: Sized 約束。
