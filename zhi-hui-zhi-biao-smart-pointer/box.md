# Box

## 簡介

在堆積(heap)上配置空間儲存資料。

* 指向堆積上的資料，Box變數本身配置在堆疊上，佔 1 usize 的空間。
* 保有內容的所有權，Box 生命週期結束會 drop 它的資料。
* 是最泛用的智慧指標。 類似 C++的`std::unique_ptr`。
* box 只提供了間接存儲和堆分配；他們並沒有任何其他特殊的功能，比如我們將會見到的其他智慧指針。它們也沒有這些特殊功能帶來的效能損失。

```rust
fn main() {
    // 將值放入box
    let val = 5u8;
    let boxed = Box::new(val);

    // 解引用
    let val2 = *boxed;
    println!("val2={val2}"); //5

    // 自動解引用
    implicit_deref(&boxed); // 5
}
fn implicit_deref(a: &u8) {
    println!("{a}");
}
```

### 什麼時候該用 Box

* 你需要儲存遞迴的資料，且無法靜態(在編譯期)決定型別大小，如在樹中需須遞迴建立節點。
* 你需要轉移資料的所有權但想避免複製整個物件。
* 你需要將資料配置在 heap 上。
* 你需要一個 null pointer（Option\<Box>）。
* 你想寫簡單的 singly linked list。
* 你需要做 dynamic dispatch，例如 dyn Trait（former Trait Object）。

## 範例：無法在編譯期決定長度

```rust
#[derive(Debug)]
enum List {
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
