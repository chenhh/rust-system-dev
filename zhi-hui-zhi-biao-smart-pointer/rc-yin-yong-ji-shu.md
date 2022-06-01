# Rc引用計數

## 簡介

為了啟用多所有權，Rust 有一個叫做 `Rc` 的類型。其名稱為 引用計數（reference counting）的縮寫。引用計數意味著記錄一個值引用的數量來知曉這個值是否仍在被使用。如果某個值有零個引用，就代表沒有任何有效引用並可以被清理。

<mark style="color:blue;">Rc 用於當我們希望在堆上分配一些內存供程式的多個部分讀取，而且無法在編譯時確定程式的哪一部分會最後結束使用它的時候</mark>。如果確實知道哪部分是最後一個結束使用的話，就可以令其成為數據的所有者，正常的所有權規則就可以在編譯時生效。

注意 Rc 只能用於單執行緒場景；

## Rc (reference counting)

單執行緒的引用計數指標，用來共享配置在堆積上的資料。

* 內部記錄引用資料的指標數量，當最後一個 Rc 指標銷毀時，資料也隨風而去。
* &#x20;因為共享，所以禁止任何改變（但可從內部可變性繞過）。&#x20;
* 可能發生迴圈引用，導致記憶體洩漏，此時須使用 Weak 弱引用。&#x20;
* <mark style="color:red;">非原子操作，所以無法在執行緒間傳遞</mark>，但可用 Arc 原子引用計數指標。&#x20;
* 類似 C++std::shared\_ptr。

![b,c共享a的所有權](../../.gitbook/assets/rc-min.PNG)

```rust
#[derive(Debug)]
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use std::rc::Rc;
use List::{Cons, Nil};

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a)); // 1
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a)); // 2

    let c = Cons(4, Rc::clone(&a));
    println!("count after creating c = {}", Rc::strong_count(&a));// 3

    println!("{:?}", a); // Cons(5, Cons(10, Nil))
    println!("{:?}", b); // Cons(3, Cons(5, Cons(10, Nil)))
    println!("{:?}", c); // Cons(4, Cons(5, Cons(10, Nil)))
}
```

`Rc::clone` 的實現並不像大部分類型的 clone 實現那樣對所有數據進行深拷貝。`Rc::clone` 只會增加引用計數，這並不會花費多少時間。

在程式中每個引用計數變化的點，會列印出引用計數，其值可以通過調用 `Rc::strong_count` 函數獲得。這個函數叫做 strong\_count 而不是 count 是因為 Rc 也有 `weak_count`。

```rust
pub struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<T>,
}
struct RcBox<T: ?Sized> {
    strong: Cell<usize>,
    weak: Cell<usize>,
    value: T,
}
```

建立 Rc 並共享引用（複製 Rc pointer）。

```rust
use std::rc::Rc;
fn main() {
    let obj = Rc::new((1, 2, 3));
    let another_obj = Rc::clone(&obj);
    assert_eq!(obj, another_obj);
}
```

### 什麼時候該用 Rc

* 你需要共享一堆引用，但不確定哪個引用的生命週期會先結束。
* 你的資源不足，但需要一個 GC（Rc + RefCell = 窮人的 GC）。

## 引用循環與記憶體洩漏

Rust 的記憶體安全性保證使其難以意外地製造永遠也不會被清理的記憶體，稱為記憶體洩漏（memory leak）），但這並不是不可能。與在編譯時拒絕資料競爭不同， <mark style="color:red;">Rust 並不保證完全地避免記憶體洩漏，這意味著記憶體洩漏在 Rust 被認為是記憶體安全的</mark>。

這一點可以通過 `Rc` 和 `RefCell` 看出：創建引用循環的可能性是存在的。這會造成記憶體洩漏，因為每一項的引用計數永遠也到不了 0，其值也永遠不會被丟棄。

## 製造引用循環

這個特定的例子中，創建了引用循環之後程式立刻就結束了。這個循環的結果並不可怕。如果在更為復雜的程式中並在循環裡分配了很多記憶體並佔有很長時間，這個程式會使用多於它所需要的記憶體，並有可能壓垮系統並造成沒有記憶體可供使用。

創建引用循環並不容易，但也不是不可能。。創建引用循環是一個程式上的邏輯 bug，你應該使用自動化測試、或其他軟體開發最佳實踐來使其最小化。

另一個解決方案是重新組織資料結構，使得一部分引用擁有所有權而另一部分沒有。換句話說，循環將由一些擁有所有權的關係和一些無所有權的關係組成，只有所有權關系才能影響值是否可以被丟棄。

```rust
use crate::List::{Cons, Nil};
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}

fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));

    println!("a initial rc count = {}", Rc::strong_count(&a)); // 1
    println!("a next item = {:?}", a.tail()); // Some(RefCell { value: Nil })

    // b指向a
    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a)))); 

    println!("a rc count after b creation = {}", Rc::strong_count(&a)); // 2
    println!("b initial rc count = {}", Rc::strong_count(&b)); // 1
    // Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
    println!("b next item = {:?}", b.tail()); // 

    // 將a指向b
    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("b rc count after changing a = {}", Rc::strong_count(&b)); // 2
    println!("a rc count after changing a = {}", Rc::strong_count(&a)); // 2

    // Uncomment the next line to see that we have a cycle;
    // it will overflow the stack
    // println!("a next item = {:?}", a.tail());
}
```

## 避免引用循環：將 Rc 變為 Weak

們已經展示了呼叫 `Rc::clone` 會增加 `Rc` 實例的 `strong_count`，和只在其 `strong_count` 為 0 時才會被清理的 `Rc` 實例。

## 升級 `Weak` -> `Rc` ，與降級 `Rc` -> `Weak`

你也可以通過調用 Rc::downgrade 並傳遞 Rc 實例的引用來創建其值的弱引用（weak reference）。調用 Rc::downgrade 時會得到 Weak 類型的智慧指`標`。不同於將 Rc 實例的 strong\_count 加1，調用 Rc::downgrade 會將 weak\_count 加1。

Rc 類型使用 weak\_count 來記錄其存在多少個 Weak 引用，類似於 strong\_count。<mark style="color:red;">其區別在於 weak\_count 無需計數為 0 就能使 Rc 實例被清理</mark>。

強引用代表如何共享 Rc 實例的所有權，<mark style="color:red;">但弱引用並不屬於所有權關係</mark>。他們不會造成引用循環，因為任何弱引用的循環會在其相關的強引用計數為 0 時被打斷。

因為 Weak 引用的值可能已經被丟棄了，為了使用 Weak 所指向的值，我們必須確保其值仍然有效。為此可以調用 Weak 實例的 upgrade 方法，這會返回 Option\<Rc>。如果 Rc 值還未被丟棄，則結果是 Some；如果 Rc 已被丟棄，則結果是 None。

```rust
use std::rc::Rc;
fn main() {
    let five = Rc::new(5);
    // Rc -> Weak
    let weak_five = Rc::downgrade(&five);
    println!("{}, {}", Rc::strong_count(&five), Rc::weak_count(&five));//1, 1
    // Waek -> Rc
    let strong_five: Option<Rc<_>> = weak_five.upgrade();
    assert!(strong_five.is_some());
    // Destroy all strong pointers.
    drop(strong_five);
    drop(five);
    assert!(weak_five.upgrade().is_none());
}
```

### Rc 如何管理 weak 與 strong reference

```rust
unsafe impl<#[may_dangle] T: ?Sized> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            self.dec_strong();
            if self.strong() == 0 {
                // destroy the contained object
                ptr::drop_in_place(self.ptr.as_mut());
                // remove the implicit "strong weak" pointer now that we've
                // destroyed the contents.
                self.dec_weak();
                if self.weak() == 0 {
                    Global.dealloc(self.ptr.cast(), Layout::for_value(self.ptr.as_ref()));
                }
            }
        }
    }
}
```

###
