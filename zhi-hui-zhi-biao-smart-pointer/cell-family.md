# Cell, Refcell, OnceCell

## 簡介

Rust 記憶體安全基於以下規則：給定一個物件 `T`，它只能具有以下之使用情境：

* 物件具有多個不可變引用 (`&T`) (也稱為別名)。
* 物件有一個可變引用 (`&mut T`) (也稱為可變性)。

這由 Rust 編譯器強制執行。但是，在某些情況下，此規則不夠靈活。有時需要對一個物件進行多次引用，然後對其進行可變的操作。

[`Cell<T>`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html)、[`RefCell<T>`](https://rustwiki.org/zh-CN/std/cell/struct.RefCell.html) 和 [`OnceCell<T>`](https://rustwiki.org/zh-CN/std/cell/struct.OnceCell.html) 允許以單執行緒方式實現內部可變性，但它們不實現 [`Sync`](https://rustwiki.org/zh-CN/std/marker/trait.Sync.html)。

內部可變性（Interior mutability）是 Rust 中的一個設計模式，<mark style="color:red;">它允許你即使在有不可變引用時也可以改變資料</mark>，如不可變引用中的struct中有部份欄位是可變的，這通常是借用規則所不允許的。

```rust
fn main() {
    let x = 5;
    // error
    // cannot borrow `x` as mutable, as it is not declared as mutable
    let y = &mut x;
}
```

為了改變資料，該模式在資料結構中使用 unsafe 代碼來模糊 Rust 通常的可變性和借用規則。

RefCell 只能用於單執行緒場景;

所有權、借用規則與這些智慧指標做一個對比：

| Rust 規則            | 智慧指標帶來的額外規則            |
| ------------------ | ---------------------- |
| 一個資料只有一個所有者        | Rc/Arc讓一個資料可以擁有多個所有者   |
| 多個不可變借用不能和一個可變借用共存 | RefCell實現編譯期可變、不可變引用共存 |
| 違背規則導致編譯錯誤         | 違背規則導致執行階段panic        |

### 選擇 `Box<T>`、`Rc<T>` 或 `RefCell<T>` 的時機

* `Rc<T>` 讓數個擁有者能共享(唯讀)相同資料；`Box<T>` 與 `RefCell<T>` 只能有一個擁有者。
* `Box<T>` 能有不可變或可變的借用並在編譯時檢查；`Rc<T>` 則只能有不可變借用並在編譯時檢查：`RefCell<T>` 能有不可變或可變借用但是在執行時檢查。
* 由於 `RefCell<T>` 允許在執行時檢查可變參考，你可以改變 `RefCell<T>` 內部的數值，就算 `RefCell<T>` 是不可變的。

### 什麼時候該用 Cell 或 RefCell

* 你需要一個對外不可變，但內部狀態可變的資料結構。
* 你需要動態建立多個可變的 alias。
* 你想要變更 Rc 多個指標底層的資料狀態。

## Cell

[https://rustwiki.org/zh-CN/std/cell/index.html](https://rustwiki.org/zh-CN/std/cell/index.html)

Cell 和 RefCell 在功能上沒有區別，區別在於 Cell 適用於 `T` 實現 Copy 的情形。

[`Cell<T>`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html) 通過將值移入和移出 cell 來實現內部可變性。由於 Cell 型別針對的是實現了 Copy trait的值類型，因此在實際開發中，Cell 使用的並不多。

<mark style="color:red;">總之，當非要使用內部可變性時，首選 Cell，只有你的類型沒有實現 Copy 時，才去選擇 RefCell</mark>。

該類型提供了以下方法:

* 對於實現 [`Copy`](https://rustwiki.org/zh-CN/std/marker/trait.Copy.html) 的類型，[`get`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html#method.get) 方法通過複製它來檢索當前內部值。
* 對於實現 [`Default`](https://rustwiki.org/zh-CN/std/default/trait.Default.html) 的類型，[`take`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html#method.take) 方法將當前內部值取代為 [`Default::default()`](https://rustwiki.org/zh-CN/std/default/trait.Default.html#tymethod.default)，然後返回替換後的值。
* 所有類型都有:
  * [`replace`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html#method.replace): 替換當前內部值並返回替換後的值。
  * [`into_inner`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html#method.into\_inner): 此方法使用 `Cell<T>` 並返回內部值。
  * [`set`](https://rustwiki.org/zh-CN/std/cell/struct.Cell.html#method.set): 該方法替換內部值，丟棄替換後的值。

```rust
use std::cell::Cell;
fn main() {
    // hello world是 &str 型別，它實現了 Copy trait，但不可用於String類型
    let c = Cell::new("hello world");
    // c.get 用來取值，c.set 用來設定新值
    let one = c.get();    // 傳回內部值的copy
    c.set("kkkk");
    let two = c.get();
    println!("{one}:{two}"); // hello world:kkkk
}
```

```rust
use std:: cell:: Cell;

fn main() {
    let data : Cell< i32> = Cell:: new(100);
    let p = &data;
    data.set(10);

    println!("{}", p.get());    //10
    p.set(20);
    println!("{:?}", data); //Cell { value: 20 }
}
```

```rust
// Cell 沒有額外的效能損耗，例如以下兩段程式碼的效能其實是一致
// code snipet 1 (可成功編譯)
let x = Cell::new(1);
let y = &x;
let z = &x;
x.set(2);
y.set(3);
z.set(4);
println!("{}", x.get());

// code snipet 2 (無法編譯)
let mut x = 1;
let y = &mut x;
let z = &mut x;
x = 2;
*y = 3;
*z = 4;
println!("{}", x);
```

## 通過 RefCell 在執行時檢查借用規則

不同於`Rc`(有一個以上的所有者)，`RefCell` 代表其資料的**唯一**的所有權。

**而**`RefCell`與`Box`都有唯有唯一的擁有權，其不同之處在於<mark style="color:red;">對於引用和 Box，借用規則的不可變性作用於編譯時</mark>。<mark style="color:red;">對於 RefCell，這些不可變性作用於執行時</mark>。

對於引用，如果違反這些規則，會得到一個編譯錯誤。而對於 `RefCell`，如果違反這些規則程式會 panic 並退出。

RefCell 是會動態檢查 borrow checker rule 的型別，比 Cell 多一個 borrow 的欄位，動態紀錄引用的情形。

<mark style="color:red;">RefCell 實際上並沒有解決可變引用和引用可以共存的問題，只是將報錯從編譯期推遲到執行階段，從編譯器錯誤變成了 panic 異常</mark>。

RefCell 的特點：

* 在不確定一個對像是否實現了 Copy 時，直接選 RefCell；
* 如果被包裹對象，`可變借用`兩次會導致執行緒崩潰。所以需要使用者自行判斷；
* RefCell 只能用於執行緒內部，不能跨執行緒；
* RefCell 常常與 Rc 配合使用（都是單執行緒內部使用）；

當呼叫這些函數時，它們首先驗證是否滿足 Rust 的借用規則: 允許任意數量的不可改變借用或允許單個不可改變借用，但絕不能同時使用。 如果嘗試違反這些規則的借用，執行緒將崩潰。

```rust
use std::cell::RefCell;

fn main() {
    let s = RefCell::new(String::from("hello, world"));
    let s1 = s.borrow();
    let s2 = s.borrow_mut();    //panic, already borrowed: BorrowMutError

    println!("{s1},{s2}");
}
```

可以使用 [`borrow`](https://rustwiki.org/zh-CN/std/cell/struct.RefCell.html#method.borrow) 獲取對 `RefCell` 的內部值 (`&T`) 的不可更改引用。

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(5);

    let borrowed_five = c.borrow();
    let borrowed_five2 = c.borrow();

    // RefCell { value: 5 }, 5, 5
    println!("{c:?}, {borrowed_five:?}, {borrowed_five2:?}");
}

```

使用 [`borrow_mut`](https://rustwiki.org/zh-CN/std/cell/struct.RefCell.html#method.borrow\_mut) 可以獲取可更改引用 (`&mut T`)。&#x20;

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(5);
    // // 以可寫的方式 borrow
    *c.borrow_mut() = 10;

    // RefCell { value: 10 }
    println!("{c:?}");
}

```



### RefCell實作

```rust
pub struct RefCell<T: ?Sized> {
    borrow: Cell<BorrowFlag>, // 
    value: UnsafeCell<T>,
}
```

當創建不可變和可變引用時，我們分別使用 `&` 和 `&mut` 語法。對於 RefCell 來說，則是 `borrow` 和 `borrow_mut` 方法，這屬於 RefCell 安全 API 的一部分。`borrow` 方法返回 `Ref` 類型的智慧指針，`borrow_mut` 方法返回 `RefMut` 類型的智慧指針。這兩個類型都實現了 `Deref`，所以可以當作常規引用對待。

```rust
use std::cell::RefCell;
fn main() {
    let c = RefCell::new(5); // create a RefCell
    *c.borrow_mut() += 5; // mutably borrow and mutate interior value
    assert_eq!(&*c.borrow(), &10); // immutably borrow to check the value
}
```

`RefCell` 記錄當前有多少個活動的 `Ref` 和 RefMut 智慧指針。每次調用 `borrow`，`RefCell` 將活動的不可變借用計數加一。當 `Ref` 值離開作用域時，不可變借用計數減一。就像編譯時借用規則一樣，`RefCell` 在任何時候只允許有多個不可變借用或一個可變借用。

在運行時捕獲借用錯誤而不是編譯時意味著將會在開發過程的後期才會發現錯誤，甚至有可能發布到生產環境才發現；還會因為在運行時而不是編譯時記錄借用而導致少量的運行時效能懲罰。



## 結合 Rc 和 RefCell 來擁有多個可變資料所有者

回憶一下 `Rc` 允許對相同資料有多個所有者，不過只能提供資料的不可變訪問。如果有一個儲存了 `RefCell` 的 `Rc` 的話，就可以得到有多個所有者並且可以修改的值了。

```rust
use std::cell::RefCell;
use std::rc::Rc;
fn main() {
    let s = Rc::new(RefCell::new("我很善變，還擁有多個主人".to_string()));

    let s1 = s.clone();
    let s2 = s.clone();
    // let mut s2 = s.borrow_mut();
    // 指向同一物件，因此改變一個全部指標指向內容都會變
    s2.borrow_mut().push_str(", oh yeah!");

    println!("{:?}\n{:?}\n{:?}", s, s1, s2);
}

/*
RefCell { value: "我很善變，還擁有多個主人, oh yeah!" }
RefCell { value: "我很善變，還擁有多個主人, oh yeah!" }
RefCell { value: "我很善變，還擁有多個主人, oh yeah!" }
*/

```

```rust
// Rc<T> 只存放不可變值，所以一旦創建了這些列表值後就不能修改。
// 讓我們加入 RefCell<T> 來獲得修改列表中值的能力
#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));

    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    // 修改list最後的值
    // 解引用 Rc<T> 以獲取其內部的 RefCell<T> 值。
    // borrow_mut 方法返回 RefMut<T> 智慧指標
    *value.borrow_mut() += 10;
    
    // a after = Cons(RefCell { value: 15 }, Nil)
    println!("a after = {:?}", a);
    // b after = Cons(RefCell { value: 3 }, Cons(RefCell { value: 15 }, Nil))  
    println!("b after = {:?}", b);
    // c after = Cons(RefCell { value: 4 }, Cons(RefCell { value: 15 }, Nil))  
    println!("c after = {:?}", c);  
}
```

通過使用 `RefCell`，我們可以擁有一個表面上不可變的 List，不過可以使用 `RefCell` 中提供內部可變性的方法來在需要時修改資料。`RefCell` 的運行時借用規則檢查也確實保護我們免於出現資料競爭——有時為了資料結構的靈活性而付出一些效能是值得的。

## Cell與RefCell

Cell 透過所有權轉移修改內部的值。 RefCell 藉由執行時的借用檢查操作指標。 兩者皆不會額外在堆積配置空間存資料。

### Cell實作

Cell 的結構體不包含其他 metadata，直接就是透過 UnsafeCell 儲存值。

```rust
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```

Cell 對不同型別的操作維持一個共識：「轉移整個資料的所有權」。例如 get、set 皆是直接 copy 或是取代原本的值。

```rust
use std::cell::Cell;
fn main() {
    let c = Cell::new(5);
    println!("{:?}", c);    // 5
    c.set(10); // discard 5 and value set to 10
    println!("{:?}", c);    //10
    c.get(); // 10 (a copy but not a reference)
    c.replace(7); // 10 (returns the replaced value)
    println!("{:?}", c);    // 7   
}
```

## OnceCell\<T>

[`OnceCell<T>`](https://rustwiki.org/zh-CN/std/cell/struct.OnceCell.html) 在某種程度上是 `Cell` 和 `RefCell` 的混合體，適用於通常只需要設定一次的值。 這意味著無需移動或複製內部值 (與 `Cell` 不同) 也無需執行階段檢查 (與 `RefCell` 不同) 即可獲得引用 `&T`。 然而，它的值一旦設定也不能更新，除非您有一個可變引用到 `OnceCell`。
