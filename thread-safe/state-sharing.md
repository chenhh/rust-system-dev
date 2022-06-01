# 狀態共享

## 簡介

Rust的設計一方面禁止了我們在執行緒之間隨意共用變數，另一方面提供了一些工具類型供我們使用，使我們可以安全地在執行緒之間共用變數。

Rust之所以這麼設計，是因為設計者觀察到了發生“資料競爭”的根源是什麼。簡單總結就是：Alias+Mutation+No ordering。

實際上我們可以看到，<mark style="color:red;">Rust保證記憶體安全的思路和執行緒安全的思路是一致的。</mark>在多執行緒中，我們要保證沒有資料競爭，一般是通過下面的方式：

1. <mark style="background-color:red;">多個執行緒可以同時讀共用變數；(共享不可變)</mark>。
2. <mark style="background-color:red;">只要存在一個執行緒在寫共用變數，則不允許其他執行緒讀/寫共用變數。(可變不共享)</mark>。

這和記憶體安全的設計實際上是相同的。如果沒有“預設記憶體安全”打下的良好基礎，Rust就沒辦法做到“執行緒安全”；正因為在“記憶體安全”問題上的一系列基礎性設計，才導致了“執行緒安全”基本就是水到渠成的結果。

Rust的這套執行緒安全設計有以下好處：

* 免疫一切數據競爭；·無額外性能損耗；
* 無須與編譯器緊耦合。

我們可以觀察到一個有趣的現象：Rust語言實際上並不知曉“執行緒”這個概念，相關類型都是寫在標準庫中的，與其他類型並無二致。<mark style="background-color:orange;">**Rust語言提供的僅僅只是Sync、Send這樣的一般性概念，以及生命週期分析、“borrow check”分析這樣的機制。Rust編譯器本身並未與“執行緒安全”“資料競爭”等概念深度綁定，也不需要一個runtime來輔助完成功能**</mark>。然而，通過這些基本概念和機制，它卻實現了完全通過編譯階段靜態檢查實現“免除資料競爭”這樣的目標。

## Arc

Arc是Rc的執行緒安全版本。它的全稱是“Atomic reference counter”。注意第一個單詞代表的是atomic而不是automatic。它強調的是“原子性”。它跟Rc最大的區別在於，引用計數用的是原子整數類型。

```rust
use std::sync::Arc;
use std::thread;
fn main() {
    // 由編譯器決定Vec中的類型T
    let numbers: Vec<_> = (0..100u32).collect();
    // 引用計數指標,指向一個 Vec
    let shared_numbers = Arc::new(numbers);
    // 迴圈創建 10 個執行緒
    for _ in 0..10 {
        // 複製引用計數指標,所有的 Arc 都指向同一個 Vec
        let child_numbers = shared_numbers.clone();
        // move修飾閉包,上面這個 Arc 指標被 move 進入了新執行緒中
        thread::spawn(move || {
            // 我們可以在新執行緒中使用 Arc,讀取共用的那個 Vec
            let local_numbers = &child_numbers[..];
            // 繼續使用 Vec 中的資料
        });
    }
}
```

如果不小心把Rc用在了多執行緒環境，直接是編譯錯誤，根本不會引發多執行緒同步的問題。如果不小心把Arc用在了單執行緒環境也沒什麼問題，不會有bug出現，只是引用計數增加或減少的時候效率稍微有一點降低。

## mutex

根據Rust的“共用不可變，可變不共用”原則，Arc既然提供了共用引用，就一定不能提供可變性。所以，Arc也是唯讀的，它對外API和Rc是一致的。如果我們要修改怎麼辦？同樣需要“內部可變性”。這種時候，我們需要用執行緒安全版本的“內部可變性”，如Mutex和RwLock。

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;
const COUNT: u32 = 1000000;
fn main() {
    let global = Arc::new(Mutex::new(0));
    // 為了避免把global這個引用計數指標move進入閉包，
    // 所以在外面先提前複製一份，然後將複製出來的這個指標傳入閉包中
    let clone1 = global.clone();
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone1.lock().unwrap();
            *value += 1;
        }
    });
    let clone2 = global.clone();
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone2.lock().unwrap();
            *value -= 1;
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```

如果當前Mutex已經是“有毒”（Poison）的狀態，它返回的就是錯誤。什麼情況會導致Mutex有毒呢？當Mutex在鎖住的同時發生了panic，就會將這個Mutex置為“有毒”的狀態，以後再調用lock（）都會失敗。這個設計是為了panic safety而考慮的，主要就是考慮到在鎖住的時候發生panic可能導致Mutex內部資料發生混亂。所以這個設計防止再次進入Mutex內部的時候訪問了被破壞掉的資料內容。如果有需要的話，使用者依然可以手動調用PoisonError::into\_inner（）方法獲得內部資料。

而MutexGuard類型則是一個“智慧指標”類型，它實現了DerefMut和Deref這兩個trait，所以它可以被當作指向內部資料的普通指標使用。MutexGuard實現了一個解構函數，通過RAII手法，在解構函數中調用了unlock（）方法解鎖。因此，使用者是不需要手動調用方法解鎖的。

Rust的這個設計，優點不在於它“允許你做什麼”，而在於它“不允許你做什麼”。

* 如果我們誤用了Rc\<isize>來實現執行緒之間的共用，就是編譯錯誤。根據編譯錯誤，
* 我們將指標改為Arc類型，然後又會發現，它根本沒有提供可變性。它的API只能共用讀，根本沒有寫資料的方法存在。
* 此時，我們會想到加入內部可變性來允許多執行緒共用讀寫。如果我們使用了Arc\<RefCell<\_>>類型，依然是編譯錯誤。因為RefCell類型不滿足Sync。而Arc\<T>需要內部的T參數必須滿足T：Sync，才能使Arc滿足Sync。

把這些綜合起來，我們可以推理出Arc\<RefCell<\_>>是！Sync。最終，編譯器把其他的路都堵死了，唯一可以編譯通過的就是使用那些滿足Sync條件的類型，比如Arc\<Mutex<\_>>。在使用的時候，我們也不可能忘記調用lock方法，因為Mutex把真實資料包裹起來了，只有調用lock方法才有機會訪問內部資料。我們也不需要記得調用unlock方法，因為lock方法返回的是一個MutexGuard類型，這個類型在解構的時候會自動調用unlock。所以，編譯器在逼著使用者用正確的方式寫程式碼。

## RwLock

RwLock就是“讀寫鎖”。它跟Mutex很像，主要區別是對外暴露的API不一樣。對Mutex內部的資料讀寫，RwLock都是調用同樣的lock方法；而對RwLock內部的資料讀寫，它分別提供了一個成員方法read/write來做這個事情。其他方面基本和Mutex一致。

```rust
use std::sync::Arc;
use std::sync::RwLock;
use std::thread;
const COUNT: u32 = 1000000;
fn main() {
    let global = Arc::new(RwLock::new(0));
    let clone1 = global.clone();
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone1.write().unwrap();
            *value += 1;
        }
    });
    let clone2 = global.clone();
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone2.write().unwrap();
            *value -= 1;
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```

## Atomic

Rust標準庫還為我們提供了一系列的“原子操作”資料類型，它們在std::sync::atomic模組裡面。它們都是符合Sync的，可以在多執行緒之間共用。比如，我們有AtomicIsize類型，顧名思義，它對應的是isize類型的“執行緒安全”版本。我們知道，普通的整數讀取再寫入，這種操作是非原子的。而原子整數的特點是，可以把“讀取”“計算”“再寫入”這樣的操作編譯為特殊的CPU指令，保證這個過程是原子操作。

```rust
use std::sync::atomic::{AtomicIsize, Ordering};
use std::thread;
use std::sync::Arc;
const COUNT: u32 = 1000000;
fn main() {
    // Atomic 系列類型同樣提供了執行緒安全版本的內部可變性
    let global = Arc::new(AtomicIsize::new(0));
    let clone1 = global.clone();
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            clone1.fetch_add(1, Ordering::SeqCst);
        }
    });
    let clone2 = global.clone();
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            clone2.fetch_sub(1, Ordering::SeqCst);
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```

## Deadlock

假設有5個哲學家，共用一張放有5把椅子的桌子，每人分得一把椅子，但是，桌子上共有5支筷子，在每人兩邊各放一支，哲學家們在肚子饑餓時才試圖分兩次從兩邊拿起筷子就餐。條件:·拿到兩支筷子時哲學家才開始吃飯；·如果筷子已在他人手上，則該哲學家必須等他人吃完之後才能拿到筷子；·任一哲學家在自己未拿到兩隻筷子前卻不放下自己手中的筷子。

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;
struct Philosopher {
    name: String,
    left: usize,
    right: usize,
}
impl Philosopher {
    fn new(name: &str, left: usize, right: usize) -> Philosopher {
        Philosopher {
            name: name.to_string(),
            left: left,
            right: right,
        }
    }
    fn eat(&self, table: &Table) {
        let _left = table.forks[self.left].lock().unwrap();
        println!("{} take left fork.", self.name);
        thread::sleep(Duration::from_secs(2));
        let _right = table.forks[self.right].lock().unwrap();
        println!("{} take right fork.", self.name);
        thread::sleep(Duration::from_secs(1));
        println!("{} is done eating.", self.name);
    }
}
struct Table {
    forks: Vec<Mutex<()>>,
}
fn main() {
    let table = Arc::new(Table {
        forks: vec![
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
        ],
    });
    let philosophers = vec![
        Philosopher::new("Judith Butler", 0, 1),
        Philosopher::new("Gilles Deleuze", 1, 2),
        Philosopher::new("Karl Marx", 2, 3),
        Philosopher::new("Emma Goldman", 3, 4),
        Philosopher::new("Michel Foucault", 4, 0),
    ];
    let handles: Vec<_> = philosophers
        .into_iter()
        .map(|p| {
            let table = table.clone();
            thread::spawn(move || {
                p.eat(&table);
            })
        })
        .collect();
    for h in handles {
        h.join().unwrap();
    }
}
```

我們可以發現，5個哲學家都拿到了他左邊的那支筷子，而都在等待他右邊的那支筷子。在沒等到右邊筷子的時候，每個人都不會釋放自己已經拿到的那支筷子。於是，大家都進入了無限的等待之中，程式無法繼續執行了。這就是“鎖死”。在Rust中，“鎖死”問題是沒有辦法在編譯階段由靜態檢查來解決的。就像前面提到的“迴圈引用製造記憶體洩漏”一樣，編譯器無法通過靜態檢查來完全避免這個問題，需要程式設計師自己注意。

## Barrier

Barrier是這樣的一個類型，它使用一個整數做初始化，可以使得多個執行緒在某個點上一起等待，然後再繼續執行。

```rust
use std::sync::{Arc, Barrier};
use std::thread;
fn main() {
    let barrier = Arc::new(Barrier::new(10));
    let mut handlers = vec![];
    for _ in 0..10 {
        let c = barrier.clone();
        // The same messages will be printed together.
        // You will NOT see any interleaving.
        let t = thread::spawn(move || {
            println!("before wait");
            c.wait();
            println!("after wait");
        });
        handlers.push(t);
    }
    for h in handlers {
        h.join().ok();
    }
}
```

這個程式創建了一個多個執行緒之間共用的Barrier，它的初始值是10。我們創建了10個子執行緒，每個子執行緒都有一個Arc指標指向了這個Barrier，並在子執行緒中調用了Barrier：：wait方法。這些子執行緒執行到wait方法的時候，就開始進入等候狀態，一直到wait方法被調用了10次，10個子執行緒都進入等候狀態，此時Barrier就通知這些執行緒可以繼續了。然後它們再開始執行下面的邏輯。所以最終的執行結果是：先列印出10條before wait，再列印出10條after wait，絕不會錯亂。

## Condvar

Condvar是條件變數，它可以用於等待某個事件的發生。在等待的時候，這個執行緒處於阻塞狀態，並不消耗CPU資源。在常見的作業系統上，Condvar的內部實現是調用的作業系統提供的條件變數。它調用wait方法的時候需要一個MutexGuard類型的參數，因此Condvar總是與Mutex配合使用的。而且我們一定要注意，一個Condvar應該總是對應一個Mutex，不可混用，否則會導致執行階段的panic。

Condvar的一個常見使用模式是和一個Mutex\<bool>類型結合使用。我們可以用Mutex中的bool變數存儲一個舊的狀態，在條件發生改變的時候修改它的狀態。通過這個狀態值，我們可以決定是否需要執行等待事件的操作。

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;
fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone();
    thread::spawn(move || {
        thread::sleep(Duration::from_secs(1));
        let &(ref lock, ref cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
        *started = true;
        cvar.notify_one();
        println!("child thread {}", *started);
    });
    // wait for the thread to start up
    let &(ref lock, ref cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    println!("before wait {}", *started);
    while !*started {
        started = cvar.wait(started).unwrap();
    }
    println!("after wait {}", *started);
}
```

這段程式碼中存在兩個執行緒之間的共用變數，包括一個Condvar和一個Mutex封裝起來的bool類型。我們用Arc類型把它們包起來。在子執行緒中，我們做完了某件工作之後，就將共用的bool類型變數設置為true，並使用Condvar：：notify\_one通知事件發生。在主執行緒中，我們首先判定這個bool變數是否為true：如果已經是true，那就沒必要進入等候狀態了；否則，就進入阻塞狀態，等待子執行緒完成任務。

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
