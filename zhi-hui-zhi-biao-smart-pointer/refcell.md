# Refcell

## 簡介

內部可變性（Interior mutability）是 Rust 中的一個設計模式，它允許你即使在有不可變引用時也可以改變資料，如不可變引用中的struct中有部份欄位是可變的，這通常是借用規則所不允許的。為了改變資料，該模式在資料結構中使用 unsafe 代碼來模糊 Rust 通常的可變性和借用規則。

RefCell 只能用於單執行緒場景;

## 通過 RefCell 在執行時檢查借用規則

不同於`Rc`，`RefCell` 代表其資料的**唯一**的所有權。

**而**`RefCell`與`Box`都有唯有唯一的擁有權，其不同之處在於<mark style="color:red;">對於引用和 Box，借用規則的不可變性作用於編譯時</mark>。<mark style="color:red;">對於 RefCell，這些不可變性作用於執行時</mark>。對於引用，如果違反這些規則，會得到一個編譯錯誤。而對於 `RefCell`，如果違反這些規則程式會 panic 並退出。

RefCell 是會動態檢查 borrow checker rule 的型別，比 Cell 多一個 borrow 的欄位，動態紀錄引用的情形。

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

## 選擇 Box，Rc 或 RefCell 的時機

* Rc 允許相同資料有多個所有者；Box 和 RefCell 有單一所有者。&#x20;
* Box 允許在編譯時執行不可變或可變借用檢查；Rc僅允許在編譯時執行不可變借用檢查；RefCell 允許在運行時執行不可變或可變借用檢查。&#x20;
* 因為 RefCell 允許在運行時執行可變借用檢查，所以我們可以在即便 RefCell 自身是不可變的情況下修改其內部的值。

### 什麼時候該用 Cell 或 RefCell

* 你需要一個對外 immutable，但內部狀態可變的資料結構。
* 你需要動態建立多個可變的 alias。
* 你想要變更 Rc 多個指標底層的資料狀態。

## 結合 Rc 和 RefCell 來擁有多個可變資料所有者

回憶一下 `Rc` 允許對相同資料有多個所有者，不過只能提供資料的不可變訪問。如果有一個儲存了 `RefCell` 的 `Rc` 的話，就可以得到有多個所有者 並且可以修改的值了！

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

&#x20;Cell 透過所有權轉移修改內部的值。 RefCell 藉由執行時的借用檢查操作指標。 兩者皆不會額外在堆積配置空間存資料。

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
