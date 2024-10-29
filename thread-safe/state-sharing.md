# 狀態共享(Arc, Mutex)

## 簡介

Rust的設計一方面禁止了我們在執行緒之間隨意共用變數，另一方面提供了一些工具類型供我們使用，使我們可以安全地在執行緒之間共用變數。

Rust之所以這麼設計，是因為設計者觀察到了發生“資料競爭”的根源是什麼。簡單總結就是：<mark style="color:red;">Alias+Mutation+No ordering</mark>。

實際上我們可以看到，<mark style="color:red;">Rust保證記憶體安全的思路和執行緒安全的思路是一致的。</mark>在多執行緒中，我們要保證沒有資料競爭，一般是通過下面的方式：

1. <mark style="background-color:red;">多個執行緒可以同時讀共用變數；(共享不可變)</mark>。
2. <mark style="background-color:red;">只要存在一個執行緒在寫共用變數，則不允許其他執行緒讀/寫共用變數。(可變不共享)</mark>。

這和記憶體安全的設計實際上是相同的。如果沒有“預設記憶體安全”打下的良好基礎，Rust就沒辦法做到“執行緒安全”；正因為在“記憶體安全”問題上的一系列基礎性設計，才導致了“執行緒安全”基本就是水到渠成的結果。

Rust的這套執行緒安全設計有以下好處：

* 免疫一切數據競爭；·無額外性能損耗；
* 無須與編譯器緊耦合。

我們可以觀察到一個有趣的現象：<mark style="color:red;">Rust語言實際上並不知曉“執行緒”這個概念</mark>，相關類型都是寫在標準庫中的，與其他類型並無二致。<mark style="background-color:orange;">**Rust語言提供的僅僅只是Sync、Send這樣的一般性概念，以及生命週期分析、“borrow check”分析這樣的機制。Rust編譯器本身並未與“執行緒安全”“資料競爭”等概念深度綁定，也不需要一個runtime來輔助完成功能**</mark>。然而，通過這些基本概念和機制，它卻實現了完全通過編譯階段靜態檢查實現“免除資料競爭”這樣的目標。

在 Rust 中有多種方式可以實現同步性。

* <mark style="color:red;">通道(channel)</mark>就像是單一所有權的模式，訊息傳進通道之後原本的訊息傳遞者就不應該再對原本的資料做操作，就像是所有權從一個變數轉移到另外一個變數一樣。我們可以通過管道來控制不同執行緒間的執行次序。
* 與此相對，<mark style="color:red;">共享記憶體(shared memory)</mark>就像是共享所有權一樣，會有複數的擁有者同時對同一塊記憶體操作，記憶體的資料會被不同執行緒更動，所以需要其他機制來管理誰可以存取這塊記憶體，其中之一就是互斥鎖Mutex。也可通過原子操作等並發原語來實現多個執行緒同時且安全地去訪問一個資源。

## Arc

[https://rustwiki.org/zh-CN/std/sync/struct.Arc.html](https://rustwiki.org/zh-CN/std/sync/struct.Arc.html)



Arc是Rc的執行緒安全版本。它跟Rc最大的區別在於，引用計數用的是原子整數類型，即在同一時間內，資料只能被一個執行緒存取，<mark style="color:red;">內部的值仍然是不可變</mark>。

```rust
use std::sync::Arc;
use std::thread;
fn main() {
    // 由編譯器決定Vec中的類型T
    let numbers: Vec<_> = (0..5u32).collect();
    // 引用計數指標,指向一個 Vec
    let shared_numbers = Arc::new(numbers);
    let mut pool = Vec::new();
    // 迴圈創建 10 個執行緒
    for idx in 0..10 {
        // 複製引用計數指標,所有的 Arc 都指向同一個 Vec
        let child_numbers = shared_numbers.clone();
        // move修飾閉包,上面這個 Arc 指標被 move 進入了新執行緒中
        let handle = thread::spawn(move || {
            // 我們可以在新執行緒中使用 Arc,讀取共用的那個 Vec
            let local_numbers = &child_numbers;
            println!("thread {idx}: vec: {:?}", local_numbers);
            // 繼續使用 Vec 中的資料
        });
        pool.push(handle);
    }
    
    for t in pool{
        t.join().expect("err");
    }
}

/*
thread 7: vec: [0, 1, 2, 3, 4]
thread 3: vec: [0, 1, 2, 3, 4]
thread 8: vec: [0, 1, 2, 3, 4]
thread 9: vec: [0, 1, 2, 3, 4]
thread 6: vec: [0, 1, 2, 3, 4]
thread 5: vec: [0, 1, 2, 3, 4]
thread 4: vec: [0, 1, 2, 3, 4]
thread 2: vec: [0, 1, 2, 3, 4]
thread 1: vec: [0, 1, 2, 3, 4]
thread 0: vec: [0, 1, 2, 3, 4]
*/
```

如果不小心把Rc用在了多執行緒環境，直接是編譯錯誤，根本不會引發多執行緒同步的問題。如果不小心把Arc用在了單執行緒環境也沒什麼問題，不會有錯誤出現，只是引用計數增加或減少的時候效率稍微有一點降低。

## mutex

```rust
#[stable(feature = "rust1", since = "1.0.0")]
#[cfg_attr(not(test), rustc_diagnostic_item = "Mutex")]
pub struct Mutex<T: ?Sized> {
    inner: sys::MovableMutex,
    poison: poison::Flag,
    data: UnsafeCell<T>,
}u
```

<mark style="background-color:orange;">單純的互斥鎖(mutex)無法在執行緒之間共享，因此必須在外層再包一個Arc提供多執行緒之間共享的方法。</mark>

互斥鎖的概念是，<mark style="color:red;">同一時間</mark><mark style="color:red;">**只允許一個執行緒存取**</mark><mark style="color:red;">特定資料(不分讀取或寫入)</mark>，互斥鎖的鎖代表的是獨佔權，只有擁有鎖的執行緒可以對特定資料存取，所以任何執行緒在操作之前都要先嘗試索取互斥鎖的鎖，如果索取失敗代表有其他執行緒正在使用，那它的操作就會被拒絕。

也因此互斥鎖有兩個重點：

1. 存取資料之前必須獲得鎖。
2. 操作完成後要釋放鎖(容易出錯)，否則其他人無法使用。

此互斥鎖將阻止等待鎖可用的執行緒。互斥鎖也可以通過 new 構造函數進行靜態初始化或創建。 每個互斥鎖都有一個類型參數，表示它正在保護的數據。 只能通過從 lock 和 try\_lock 返回的 RAII 保護來訪問數據，這保證了只有在互斥鎖被鎖定時才可以訪問數據。

根據Rust的“共用不可變，可變不共用”原則，Arc既然提供了共用引用，就一定不能提供可變性。所以，Arc也是唯讀的，它對外API和Rc是一致的。如果我們要修改怎麼辦？<mark style="color:red;">同樣需要“內部可變性”。這種時候，我們需要用執行緒安全版本的“內部可變性”，如Mutex和RwLock</mark>。

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;
const COUNT: u32 = 1000000;
fn main() {
    let global = Arc::new(Mutex::new(0));
    // 為了避免把global這個引用計數指標move進入閉包，
    // 所以在外面先提前複製一份，然後將複製出來的這個指標傳入閉包中
    let clone1 = global.clone(); // 指向同一個mutex
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            // mutex lock, 離開scope後自動unlock
            let mut value = clone1.lock().unwrap();
            *value += 1;
        }
    });
    let clone2 = global.clone();    // 指向同一個mutex
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            // mutex lock, 離開scope後自動unlock
            let mut value = clone2.lock().unwrap();
            *value -= 1;
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```

### Poisoning

此模組中的互斥鎖實現了一種稱為 “poisoning” 的策略，只要執行緒 panic 按住互斥鎖，互斥鎖就會被視為中毒。 一旦互斥鎖中毒，預設情況下，所有其他執行緒都無法訪問資料，因為它很可能已被污染 (某些不變性未得到維護)。

<mark style="color:red;">但是，中毒的互斥鎖不會阻止對底層數據的所有訪問</mark>。 PoisonError 類型具有 into\_inner 方法，該方法將返回保護，否則將在成功鎖定後返回該保護。 盡管鎖被中毒，這仍允許訪問數據。

如果當前Mutex已經是“有毒”（Poison）的狀態，它返回的就是錯誤。

什麼情況會導致Mutex有毒呢？當Mutex在鎖住的同時發生了panic，就會將這個Mutex置為“有毒”的狀態，以後再調用lock()都會失敗。這個設計是為了panic safety而考慮的，主要就是考慮到在鎖住的時候發生panic可能導致Mutex內部資料發生混亂。所以這個設計防止再次進入Mutex內部的時候訪問了被破壞掉的資料內容。如果有需要的話，使用者依然可以手動調用PoisonError::into\_inner（）方法獲得內部資料。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lock = Arc::new(Mutex::new(0_u32));
    let lock2 = Arc::clone(&lock);

    let _ = thread::spawn(move || -> () {
        // 該執行緒將首先獲取互斥鎖，因為該鎖尚未中毒，所以將解開 `lock` 的結果。
        let _guard = lock2.lock().unwrap();

        // 持有鎖時的這種 panic (`_guard` 在作用域內) 將使互斥鎖中毒。
        panic!();
    })
    .join();

    // 到此為止，鎖定都會中毒，但是可以對返回的結果進行模式匹配，以返回兩個分支上的底層防護。
    let mut guard = match lock.lock() {
        Ok(guard) => guard,
        Err(poisoned) => poisoned.into_inner(),
    };
    *guard += 1;
}
```

### MutexGuard

因為`Mutex`是一個智慧指標，准確的說是`m.lock()`返回一個智慧指針`MutexGuard`。它實現了`DerefMut`和`Deref`這兩個trait，所以它可以被當作指向內部資料的普通指標使用。`MutexGuard`實現了一個解構函數，通過RAII手法，在解構函數中調用了`unlock()`方法解鎖。因此，使用者是不需要手動調用方法解鎖的。

Rust的這個設計，優點不在於它“允許你做什麼”，而在於它“不允許你做什麼”。

* 如果我們誤用了`Rc<isize>`來實現執行緒之間的共用，就是編譯錯誤。根據編譯錯誤，
* 我們將指標改為`Arc`類型，然後又會發現，它根本沒有提供可變性。它的API只能共用讀，根本沒有寫資料的方法存在。
* 此時，我們會想到加入內部可變性來允許多執行緒共用讀寫。如果我們使用了`Arc<RefCell<_>>`類型，依然是編譯錯誤。因為RefCell類型不滿足Sync。而`Arc<T>`需要內部的T參數必須滿足T：Sync，才能使Arc滿足Sync。

把這些綜合起來，我們可以推理出Arc\<RefCell<\_>>是！Sync。最終，編譯器把其他的路都堵死了，唯一可以編譯通過的就是使用那些滿足Sync條件的類型，比如`Arc<Mutex<_>>`。在使用的時候，我們也不可能忘記調用`lock`方法，因為Mutex把真實資料包裹起來了，只有調用`lock`方法才有機會訪問內部資料。我們也不需要記得調用`unlock`方法，因為`lock`方法返回的是一個`MutexGuard`類型，這個類型在解構的時候會自動調用`unlock`。所以，編譯器在逼著使用者用正確的方式寫程式碼。

##
