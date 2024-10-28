---
description: 線程
---

# 執行緒安全

## 簡介

Rust不僅在沒有自動垃圾回收（Garbage Collection）的條件下實現了記憶體安全，而且實現了執行緒安全。Rust編譯器可以在編譯階段避免所有的資料競爭（Data Race）問題。這也是Rust的核心競爭力之一。

<mark style="color:red;">Rust語言本身並不知曉“執行緒”“併發”具體是什麼，而是抽象出了一些更高級的概念Send/Sync特徵(實現此Trait的類型變數的引用可以安全在執行緒間共享。執行緒間轉移變數必須支援Send, 共享變數必須支援Sync)，用來描述類型在併發環境下的特性</mark>。

`std::thread::spawn`函數就是一個普通函數，編譯器沒有對它做任何特殊處理。它能保證執行緒安全的關鍵是，它對參數有合理的約束條件。這樣的設計使得Rust在執行緒安全方面具備非常好的擴展性。

因為執行緒是同時運行的，所以無法預先保證不同執行緒中的程式碼的執行順序。這會導致諸如此類的問題：

* <mark style="background-color:red;">競爭條件（Race conditions）</mark>，多個執行緒以不一致的順序訪問資料或資源。
* <mark style="background-color:red;">死鎖（Deadlocks）</mark>，兩個執行緒相互等待對方，這會阻止兩者繼續運行。
* 只會發生在特定情況且難以穩定重現和修復的錯誤。

## 執行緒(thread)

執行緒是作業系統能夠進行調度的最小單位，它是行程 (process)中的實際運作單位，每個行程至少包含一個以上的執行緒。

<mark style="color:red;">簡單的說，將所要執行的工作打包(通常寫成函數或閉包)，再將工作做為參數傳進執行緒中，待執行緒完成後傳回結果</mark>。

Rust 標準函式庫使用的是 <mark style="color:red;">1:1</mark> 的執行緒實作模型，也就是每一個語言產生的執行緒就是一個作業系統的執行緒。

在多核處理器越來越普及的今天，多執行緒程式設計也用得越來越廣泛。多執行緒的優勢有：

* 容易利用多核優勢；
* 比單執行緒反應更敏捷，比多進程資源分享更容易。

多執行緒程式設計在許多領域是不可或缺的。但是，多執行緒並行，非常容易引發資料競爭(同時有多個執行緒寫入同一區塊時發生)，而且還非常不容易被發現和除錯。

<mark style="color:red;">Rust標準庫中與執行緒相關的內容在</mark>[<mark style="color:red;">std::thread</mark>](https://doc.rust-lang.org/std/thread/index.html)<mark style="color:red;">模組中。Rust中的執行緒是對作業系統執行緒的直接封裝</mark>。

### spawn 建立新的執行緒，join 等待所有執行緒完成

[https://rustwiki.org/zh-CN/std/thread/fn.spawn.html](https://rustwiki.org/zh-CN/std/thread/fn.spawn.html)

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // 此範例是在1個thread中產生1~10的迴圈
    // 產生一個新執行緒，並為其返回一個 JoinHandle。
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("數字 {i} 出現在產生的執行緒中！");
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 1..5 {
        println!("數字 {i} 出現在主執行緒中！");
        thread::sleep(Duration::from_millis(1));
    }
    // 等待子執行緒完成
    handle.join().unwrap();
}
/*
如果沒有join等待子執行緒結束，因此可能出現主執行緒先完成離開。
數字 1 出現在主執行緒中！
數字 1 出現在產生的執行緒中！
數字 2 出現在主執行緒中！
數字 3 出現在主執行緒中！
數字 4 出現在主執行緒中！
*/
```

```rust
// 以下範例是產生10個threads
use std::thread;
use std::time::Duration;


fn main() {
    let mut handles = Vec::new();
    for i in 1..10 {
        // move捕捉閉包外的變數i
        handles.push(thread::spawn(move || {
            let tid = thread::current().id();
            println!("TID: {tid:?}, 數字 {i} 出現在產生的執行緒中！");
            thread::sleep(Duration::from_millis(5));
        }));
    }

    for i in 1..5 {
        println!("數字 {i} 出現在主執行緒中！");
        thread::sleep(Duration::from_millis(1));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}

/*
TID: ThreadId(3), 數字 2 出現在產生的執行緒中！
TID: ThreadId(2), 數字 1 出現在產生的執行緒中！
數字 1 出現在主執行緒中！
TID: ThreadId(4), 數字 3 出現在產生的執行緒中！
TID: ThreadId(5), 數字 4 出現在產生的執行緒中！
TID: ThreadId(6), 數字 5 出現在產生的執行緒中！
TID: ThreadId(9), 數字 8 出現在產生的執行緒中！
TID: ThreadId(10), 數字 9 出現在產生的執行緒中！
TID: ThreadId(7), 數字 6 出現在產生的執行緒中！
TID: ThreadId(8), 數字 7 出現在產生的執行緒中！
數字 2 出現在主執行緒中！
數字 3 出現在主執行緒中！
數字 4 出現在主執行緒中！
*/
```

使用Builder如果可指定堆疊大小或執行緒名稱。

[https://rustwiki.org/zh-CN/std/thread/struct.Builder.html](https://rustwiki.org/zh-CN/std/thread/struct.Builder.html)

```rust

use std::thread;
use std::time::Duration;
fn main() {
    // 使用Builder模式為child thread指定更多的訊息
    let t = thread::Builder::new()
        .name("child1".to_string())
        .spawn(
            // 新建的thread
            || {
            println!("enter child thread.");
            thread::park();    // 進入等待狀態
            println!("resume child thread");
        })
        .unwrap();
    println!("spawn a thread");
    thread::sleep(Duration::new(5, 0));
    t.thread().unpark();    // 恢復等待的狀態
    // parent等待child thread結束
    t.join();
    println!("child thread finished");
}
/*
spawn a thread
enter child thread.
resume child thread
child thread finished
*/
```
