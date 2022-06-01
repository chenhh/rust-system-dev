# 解構函數(destructor)

## 簡介

所謂“解構函數”（destructor），是與“構造函數”（constructor）相對應的概念。**“構造函數”是物件被創建的時候調用的函數，“解構函數”是物件被銷毀的時候調用的函數**。

Rust中沒有統一的“構造函數”這個語法，物件的構造是直接對每個成員進行初始化完成的，<mark style="background-color:red;">我們一般將物件的創建封裝到普通靜態函數中（通常命名為new）</mark>。

相對於構造函數，**解構函數有更重要的作用。它會在物件消亡之前由編譯器自動調用，因此特別適合承擔物件銷毀時釋放所擁有的資源的作用**。比如，Vec類型在使用的過程中，會根據情況動態申請記憶體，當變數的生命週期結束時，就會觸發該類型的解構函數的調用。

在解構函數中，我們就有機會將所擁有的記憶體釋放掉。在解構函數中，我們還可以根據需要編寫特定的邏輯，從而達到更多的目的。解構函數不僅可以用於管理記憶體資源，還能用於管理更多的其他資源，如檔案、鎖、socket等。

**在C++中，利用變數生命週期綁定資源的使用週期，已經是一種常用的程式設計慣例。此手法被稱為RAII（Resource Acquisition Is Initialization）**。在變數生命週期開始時申請資源，在變數生命週期結束時利用解構函數釋放資源，從而達到自動化管理資源的作用，很大程度上減少了資源的洩露和誤用。<mark style="background-color:red;">在Rust中編寫“解構函數”的辦法是impl</mark> [<mark style="background-color:red;">std::ops::Drop</mark>](https://doc.rust-lang.org/std/ops/trait.Drop.html)。

```rust
pub trait Drop {
    fn drop(&mut self);
}
```

Drop trait允許在物件即將消亡之時，自行調用指定程式碼。我們來寫一個自帶解構函數的類型。

```rust
struct D(i32);
impl Drop for D {
    fn drop(&mut self) {
        println!("destruct {}", self.0);
    }
}
fn main() {
    let _x = D(1);
    println!("construct 1");
    {
        let _y = D(2);
        println!("construct 2");
        println!("exit inner scope");
        // _y生命週期結束，呼叫dtor
    }
    println!("exit main function");
    // 離開主函數後，_x的生命週期結束，呼叫dtor
}

/*
construct 1
construct 2
exit inner scope
destruct 2
exit main function
destruct 1
*/
```

。變數`_y`的生存期是內部的大括弧包圍起來的作用域（scope），待這個作用域中的程式碼執行完之後，它的解構函數就被調用；變數`_x`的生存期是整個main函數包圍起來的作用域，待這個函數的最後一條語句執行完之後，它的解構函數就被調用。

**對於具有多個區域變數的情況，解構函數的調用順序是：先構造的後解構，後構造的先解構。因為區域變數存在於一個“堆疊”的結構中，要保持“先進後出”的策略**。

## 資源管理

在創建變數的時候獲取某種資源，在變數生命週期結束的時候釋放資源，是一種常見的設計模式。這裡的資源，不僅可以包括記憶體，還可以包括其他向作業系統申請的資源。比如我們經常用到的File類型，會在創建和使用的過程中向作業系統申請打開檔案，在它的解構函數中就會去釋放檔案。所以，RAII手法是比GC更通用的資源管理手段，GC只能管理記憶體，RAII可以管理各種資源。

```rust
use std::fs::File;
use std::io::Read;
fn main() {
    // 在File的解構函數中已經定義如果退出時，會自動關檔
    let f = File::open("/target/file/path");
    if f.is_err() {
        println!("file is not exist.");
        return;
    }
    let mut f = f.unwrap();
    let mut content = String::new();
    let result = f.read_to_string(&mut content);
    if result.is_err() {
        println!("read file error.");
        return;
    }
    println!("{}", result.unwrap());
}
```

再比如標準庫中的各種複雜資料結構（如Vec, LinkedList, HashMap等），它們管理了很多在堆上動態分配的記憶體。它們也是利用“解構函數”這個功能，在生命終結之前釋放了申請的記憶體空間，因此無須像C語言那樣手動調用free函數

## 主動解構(drop)

一般情況下，區域變數的生命週期是從它的聲明開始，到當前語句塊結束。然而，我們也可以手動提前結束它的生命週期。請注意，**使用者主動調用解構函數是非法的**，我們怎樣才能讓區域變數在語句塊結束前提前終止生命週期呢？辦法是調用標準庫中的[`std::mem::drop`](https://doc.rust-lang.org/std/mem/fn.drop.html)函數。

```rust
use std::mem::drop;
fn main() {
    let mut v = vec![1, 2, 3]; // <--- v的生命週期開始
    // v.drop();    // 非法的操作
    drop(v); // ---> v的生命週期結束
    v.push(4); // 錯誤的調用
}
```

標準庫中的std::mem::drop函數是Rust中最簡單的函數，因為它的實現為“空”：

```rust
#[inline]
pub fn drop<T>(_x: T) {}
```

drop函數不需要任何的函數體，只需要參數為“值傳遞”即可。**將物件的所有權移入函數中，什麼都不用做，編譯器就會自動釋放掉這個物件了**。因為這個drop函數的關鍵在於使用move語義把參數傳進來，使得變數的所有權從調用方移動到drop函數體內，**參數類型一定要是T，而不是\&T或者其他參考類型**。

函數體本身其實根本不重要，**重要的是把變數的所有權move進入這個函數體中，函式呼叫結束的時候該變數的生命週期結束，變數的解構函數會自動調用，管理的記憶體空間也會自然釋放**。這個過程完全符合前面講的生命週期、move語義，無須編譯器做特殊處理。事實上，我們完全可以自己寫一個類似的函數來實現同樣的效果，只要保證參數傳遞是move語義即可。

因此，對於Copy類型的變數，對它調用std::mem::drop函數是沒有意義的。

```rust
use std::mem::drop;
fn main() {
    let x = 1_i32;
    println!("before drop {}", x);
    // 基本類型因為有實現copy trait
    // 因此變數傳入drop時是copy而不是move
    // 因此不會解構
    drop(x);    
    println!("after drop {}", x);
}
```

變數遮蔽（Shadowing）不會導致變數生命週期提前結束，它不等同於drop。

```rust
use std::ops::Drop;
struct D(i32);
impl Drop for D {
    fn drop(&mut self) {
        println!("destructor for {}", self.0);
    }
}
fn main() {
    let x = D(1);
    println!("construct first variable");
    let x = D(2);
    println!("construct second variable");
}

/*
construct first variable
construct second variable
destructor for 2
destructor for 1
*/
```

這裡函式呼叫的順序為：

* 先創建第一個x，再創建第二個x
* 退出函數的時候，先解構第二個x，再解構第一個x。

由此可見，在第二個x出現的時候，雖然將第一個x遮蔽起來了，但是第一個x的生命週期並未結束，它依然存在，直到函數退出。這也說明了，雖然這兩個變數綁定了同一個名字，但在編譯器內部依然將它們視為兩個不同的變數。

另外還有一個小問題需要注意，那就是底線這個特殊符號。請注意：如果你用底線來綁定一個變數，那麼這個變數會當場執行解構，而不是等到當前語句塊結束的時候再執行。底線是特殊符號，不是普通識別字。

```rust
use std::ops::Drop;
struct D(i32);
impl Drop for D {
    fn drop(&mut self) {
        println!("destructor for {}", self.0);
    }
}
fn main() {
    let _x = D(1);
    let _ = D(2);   // 立即解構
    let _y = D(3);
}

/*
destructor for 2
destructor for 3
destructor for 1
*/
```

之所以是這樣的結果，是因為用底線綁定的那個變數當場就執行了解構，而其他兩個變數等到語句塊結束了才執行解構，而且解構順序和初始化順序剛好相反。所以，如果需要利用RAII實現某個變數的解構函數在退出作用域的時候完成某些功能，千萬不要用底線來綁定這個變數。

最後，要注意區分，`std::mem::drop()`函數和`std::ops::Drop::drop()`方法。

1. `std::mem::drop()`函數是一個獨立的函數，不是某個類型的成員方法，它由程式設計師主動調用，作用是使變數的生命週期提前結束；
2. `std::ops::Drop::drop()`方法是一個trait中定義的方法，當變數的生命週期結束的時候，編譯器會自動調用，手動調用是不允許的。
3. `std::mem::drop<T>（_x:T）`的參數類型是`T`，採用的是move語義；`std::ops::Drop::drop（&mut self）`的參數類型是\&mut Self，採用的是可變借用。在解構函式呼叫過程中，我們還有機會讀取或者修改此物件的屬性。

## drop與copy trait

要想實現Copy trait，類型必須滿足一定條件。**這個條件就是：如果一個類型可以使用memcpy的方式執行複製操作，且沒有記憶體安全問題，那麼它才能被允許實現Copy trait（基本類型或是基本類型的組合）**。

反過來說，所有滿足Copy trait的類型，在需要執行move語義的時候，使用memcpy複製一份副本，不刪除原件是完全不會產生安全問題的。

而此處需要強調的是，**帶有解構函數的類型都是不能滿足Copy語義的**。因為我們不能保證，對於帶解構函數的類型，使用memcpy複製一個副本一定不會有記憶體安全問題。所以對於這種情況，編譯器是直接禁止的。

```rust
use std::ops::Drop;
struct T;
// 有實現drop類型無法實現Copy
impl Drop for T {
    fn drop(&mut self) {}
}
// compile error
impl Copy for T {}
fn main() {}
```

## 解構標記

那麼就說明，變數的生命週期並不是簡單地與某個程式碼塊一致，生命週期何時結束，很可能是由執行時的條件決定的。

```rust
use std::mem::drop;
use std::ops::Drop;
struct D(&'static str);
impl Drop for D {
    fn drop(&mut self) {
        println!("destructor {}", self.0);
    }
}
// 獲取 DROP 環境變數的值,並轉換為整數
fn condition() -> Option<u32> {
    std::env::var("DROP")
        .map(|s| s.parse::<u32>().unwrap_or(0))
        .ok()
}
fn main() {
    let var = (D("first"), D("second"), D("third"));
    match condition() {
        Some(1) => drop(var.0),
        Some(2) => drop(var.1),
        Some(3) => drop(var.2),
        _ => {}
    }
    println!("main end");
}
/* 如果我們沒有設置DROP環境變數
main end
destructor first
destructor second
destructor third

設置export DROP=2
destructor second
main end
destructor first
destructor third
*/
```

前面說過，解構函數的調用是在編譯階段就確定好了的，調用解構函數是編譯器自動插入的程式碼做的。

而且示例又表明，解構函數的具體調用時機還是跟執行時的情況相關的。那麼編譯器是怎麼做到的呢？**編譯器是這樣完成這個功能的：首先判斷一個變數是否可能會在多個不同的路徑上發生解構，如果是這樣，那麼它會在當前函式呼叫棧中自動插入一個bool類型的標記，用於標記該物件的解構函數是否已經被調用**。

```rust
// 以下為偽程式碼,僅僅是示意
fn main() {
    let var = (D("first"), D("second"), D("third"));
    // 當函數中有擁有所有權的物件時,需要有解構自動標記
    let drop_flag_0 = false; // ---
    let drop_flag_1 = false; // ---
    let drop_flag_2 = false; // ---
    // 退出語句塊時,對當前block內擁有所有權的物件調用解構函數,並設置標記
    match condition() {
        Some(1) => {
            drop(var.0);
            if (!drop_flag_0) {
                // ---
                drop_flag_0 = true; // ---
            } // ---
        }
        Some(2) => {
            drop(var.1);
            if (!drop_flag_1) {
                // ---
                drop_flag_1 = true; // ---
            } // ---
        }
        Some(3) => {
            drop(var.2);
            if (!drop_flag_2) {
                // ---
                drop_flag_2 = true; // ---
            } // ---
        }
        _ => {}
    }
    println!("main end");
    // 退出語句塊時,對當前block內擁有所有權的物件調用解構函數,並設置標記
    if (!drop_flag_0) {
        // ---
        drop(var.0); // ---
        drop_flag_0 = true; // ---
    } // ---
    if (!drop_flag_1) {
        // ---
        drop(var.1); // ---
        drop_flag_1 = true; // ---
    } // ---
    if (!drop_flag_2) {
        // ---
        drop(var.2); // ---
        drop_flag_2 = true; // ---
    } // ---
}
```

編譯器生成的程式碼類似於上面的示例，可能會有細微差別。原理是在解構函數被調用的時候，就把標記設置一個狀態，在各個可能調用解構函數的地方都先判斷一下狀態再調用解構函數。這樣，編譯階段確定生命週期和執行階段根據情況調用就統一起來了。

## Question：如果在變數生命週期結束後，出現異常，變數的解構函數是否會執行?

\[todo]

