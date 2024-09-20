# Box

## 簡介

[Struct std::boxed::Box](https://doc.rust-lang.org/std/boxed/struct.Box.html)

在 Rust 中，所有值預設都是配置在堆疊(stack)上。通過建立 Box，可以把值裝箱（boxed）來使它在堆積(heap)上分配儲存資料。

* 指向堆積上的資料，Box變數本身配置在堆疊上，佔 1 usize 的空間。
* 保有內容的所有權，Box 生命週期結束會 drop 它的資料。
* 被裝箱的值可以使用`*`運算子進行解引用；這會移除掉一層裝箱。
* 是最泛用的智慧指標。 類似 C++的`std::unique_ptr`。
* box 只提供了間接存儲和堆分配；他們並沒有任何其他特殊的功能，比如我們將會見到的其他智慧指針。它們也沒有這些特殊功能帶來的效能損失。

```rust
fn implicit_deref(a: &u8) {
    println!("{a}");
}

fn main() {
    // 將值放入box
    let val = 5u8;
    let boxed = Box::new(val);

    // 解引用
    let val2 = *boxed;
    println!("val2={val2}"); //5

    // 自動解引用
    implicit_deref(&boxed); // 5
    
    // 在表示式中，我們無法自動隱式地執行 Deref 解引用操作，
    // 需要使用 * 運算子來顯式的進行解引用
    let val2 = *boxed + 3;
    println!("val2={val2}");    // 8
}
```

```rust
use std::mem;

#[allow(dead_code)]
#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

#[allow(dead_code)]
struct Rectangle {
    p1: Point,
    p2: Point,
}

fn origin() -> Point {
    Point { x: 0.0, y: 0.0 }
}

fn boxed_origin() -> Box<Point> {
    // 在堆積上分配這個點（point），並返回一個指向它的指標
    Box::new(Point { x: 0.0, y: 0.0 })
}

fn main() {
    // （所有的類型標註都不是必需的）
    // 堆疊分配的變數
    let point: Point = origin();
    let rectangle: Rectangle = Rectangle {
        p1: origin(),
        p2: Point { x: 3.0, y: 4.0 },
    };

    // 堆積分配的 rectangle（矩形）
    let boxed_rectangle: Box<Rectangle> = Box::new(Rectangle {
        p1: origin(),
        p2: origin(),
    });

    // 函數的輸出可以裝箱
    let boxed_point: Box<Point> = Box::new(origin());

    // 兩層裝箱
    let box_in_a_box: Box<Box<Point>> = Box::new(boxed_origin());

    println!(
        "Point occupies {} bytes in the stack", // 16 bytes, 有2個f64
        mem::size_of_val(&point)
    );
    println!(
        "Rectangle occupies {} bytes in the stack", // 32 bytes， 有4個f64
        mem::size_of_val(&rectangle)    
    );

    // box 的寬度就是指針寬度
    println!(
        "Boxed point occupies {} bytes in the stack",   
        mem::size_of_val(&boxed_point) // 8 bytes，usize為64-bit
    );
    println!(
        "Boxed rectangle occupies {} bytes in the stack", 
        mem::size_of_val(&boxed_rectangle) // 8 bytes，usize為64-bit
    );
    println!(
        "Boxed box occupies {} bytes in the stack", 
        mem::size_of_val(&box_in_a_box) // 8 bytes，usize為64-bit

    );

    // 將包含在 `boxed_point` 中的資料複製到 `unboxed_point`
    let unboxed_point: Point = *boxed_point;
    println!(
        "Unboxed point occupies {} bytes in the stack", // 16 bytes
        mem::size_of_val(&unboxed_point)
    );
}

```

### 什麼時候該用 Box

* 你需要儲存遞迴的資料，且無法靜態(在編譯期)決定型別大小，如在樹中需須遞迴建立節點。
* 你需要轉移資料的所有權但想避免複製整個物件，如陣列物件。
* 你需要將資料配置在 heap 上。
* 你需要一個 null pointer（Option\<Box>）。
* 你想寫簡單的 singly linked list。
* 你需要做 dynamic dispatch，例如 dyn Trait（former Trait Object）。

## 範例：無法在編譯期決定長度

```rust
#[derive(Debug)]
enum List {
    // 將動態大小類型變為 Sized 固定大小類型
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = List::Cons(
        1,
        Box::new(List::Cons(2, Box::new(List::Cons(3, Box::new(List::Nil))))),
    );
    println!("{:?}", list); // Cons(1, Cons(2, Cons(3, Nil)))
}
```

## 範例：錯誤處理

此 main 函式本來只會回傳錯誤型別 std::io::Error，但有了 Box 的話，此簽名就能允許其他錯誤型別加入 main 本體中。

dyn Error表示任何有實作Error trait的類型都可以傳入。

```rust
use std::error::Error;
use std::fs::File;

// Box<dyn Error> 是「任何種類的錯誤」
fn main() -> Result<(), Box<dyn Error>> {
    let greeting_file = File::open("hello.txt")?;
    Ok(())
}
```

## 範例：避免複制整個物件

```rust
fn main() {
    // 在stack上建立一個長度為1000的陣列
    let arr = [0;1000];
    // 將arr所有權轉移arr1，由於 `arr` 分配在stack上，
    // 因此這裡實際上是直接重新深複製了一份資料
    let arr1 = arr;

    // arr 和 arr1 都擁有各自的stack上陣列，因此不會報錯
    println!("{:?}", arr.len());
    println!("{:?}", arr1.len());

    // 在heap上建立一個長度為1000的陣列，然後使用一個智慧指標指向它
    let arr = Box::new([0;1000]);
    // 將heap上的陣列的所有權轉移給 arr1，由於資料在heap上，
    // 因此僅僅複製了智慧指標的結構體，底層資料並沒有被複製
    // 所有權順利轉移給 arr1，arr 不再擁有所有權
    let arr1 = arr;
    println!("{:?}", arr1.len());
    // 由於 arr 不再擁有底層陣列的所有權，因此下面程式碼將報錯
    // println!("{:?}", arr.len());
}
```

## 範例：trait物件

```rust
trait Draw {
    fn draw(&self);
}

struct Button {
    id: u32,
}
impl Draw for Button {
    fn draw(&self) {
        println!("這是螢幕上第{}號按鈕", self.id)
    }
}

struct Select {
    id: u32,
}

impl Draw for Select {
    fn draw(&self) {
        println!("這個選擇框很難用{}", self.id)
    }
}

fn main() {
    let elems: Vec<Box<dyn Draw>> = vec![
        Box::new(Button { id: 1 }), 
        Box::new(Select { id: 2 })
    ];

    for e in elems {
        e.draw()
    }
}
```
