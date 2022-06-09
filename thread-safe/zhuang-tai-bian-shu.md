# 狀態變數

## 全域變數

Rust中允許存在全域變數。在基本語法章節講過，使用static關鍵字修飾的變數就是全域變數。全域變數有一個特點：如果要修改全域變數，必須使用unsafe關鍵字。

這個規定顯然是有助於執行緒安全的。如果允許任何函數可以隨便讀寫全域變數的話，執行緒安全就無從談起了。只有不用mut修飾的全域變數才是安全的，因為它只能被讀取，不能被修改。

```rust
static mut G: i32 = 1;
fn main() {
    unsafe {
        G = 2;
        println!("{}", G);
    }
}
```

有些類型的變數不用mut修飾，也是可以做修改的。比如具備內部可變性的Cell等類型。我們可以試驗一下，如果有一個全域的、具備內部可變性的變數，會發生什麼情況：

```rust
use std::cell::Cell;
use std::thread;
// compile error, Cell<i32>` 
// cannot be shared between threads safely
static G: Cell<i32> = Cell::new(1);
fn f1() {
    G.set(2);
}
fn f2() {
    G.set(3);
}
fn main() {
    thread::spawn(|| f1());
    thread::spawn(|| f2());
}
```

對於上面這個例子，我們可以推理一下，現在有兩個執行緒同時修改一個全域變數，而且修改過程沒有做任何執行緒同步，這裡肯定是有執行緒安全的問題。但是，注意這裡傳遞給spawn函數的閉包，實際上沒有捕獲任何區域變數，所以，它是滿足Send條件的。在這種情況下，執行緒不安全類型並沒有直接穿越執行緒的邊界，spawn函數這裡指定的約束條件是查不出問題來的。

但是，編譯器還設置了另外一條規則，即共用又可變的全域變數必須滿足Sync約束。根據Sync的定義，滿足這個條件的全域變數顯然是執行緒安全的。因此，編譯器把這條路也堵死了，我們不可以簡單地通過全域變數共用狀態來構造出執行緒不安全的行為。對於那些滿足Sync條件且具備內部可變性的類型，比如Atomic系列類型，作為全域變數共用是完全安全且合法的。

## 執行緒局部存儲

執行緒局部（Thread Local）的意思是，**聲明的這個變數看起來是一個變數，但它實際上在每一個執行緒中分別有自己獨立的存儲位址，是不同的變數，互不干擾**。在不同執行緒中，只能看到與當前執行緒相關聯的那個副本，因此對它的讀寫無須考慮執行緒安全問題。

在Rust中，執行緒獨立存儲有兩種使用方式。

* 可以使用#\[thread\_local]attribute來實現。這個功能目前在穩定版中還不支持，只能在nightly版本中開啟#！\[feature（thread\_local）]功能才能使用。
* 可以使用thread\_local！巨集來實現。這個功能已經在穩定版中獲得支持。

用thread\_local！聲明的變數，使用的時候要用with()方法加閉包來完成。

```rust
use std::cell::RefCell;
use std::thread;
fn main() {
    thread_local! {
    static FOO: RefCell<u32> = RefCell::new(1)
    };
    FOO.with(|f| {
        println!("main thread value1 {:?}", *f.borrow());
        *f.borrow_mut() = 2;
        println!("main thread value2 {:?}", *f.borrow());
    });
    let t = thread::spawn(move || {
        FOO.with(|f| {
            println!("child thread value1 {:?}", *f.borrow());
            *f.borrow_mut() = 3;
            println!("child thread value2 {:?}", *f.borrow());
        });
    });
    t.join().ok();
    FOO.with(|f| {
        println!("main thread value3 {:?}", *f.borrow());
    });
}
/*
main thread value1 1
main thread value2 2
child thread value1 1
child thread value2 3
main thread value3 2
*/
```

在主執行緒中將FOO的值修改為2，但是進入子執行緒後，它看到的初始值依然是1。在子執行緒將FOO的值修改為3之後回到主執行緒，主執行緒看到的值還是2。這說明，在子執行緒中和主執行緒中看到的FOO其實是兩個完全獨立的變數，互不影響。
