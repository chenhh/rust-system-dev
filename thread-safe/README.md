---
description: 線程
---

# 執行緒安全

## 簡介

Rust不僅在沒有自動垃圾回收（Garbage Collection）的條件下實現了記憶體安全，而且實現了執行緒安全。Rust編譯器可以在編譯階段避免所有的資料競爭（Data Race）問題。這也是Rust的核心競爭力之一。

<mark style="color:red;">Rust語言本身並不知曉“執行緒”“併發”具體是什麼，而是抽象出了一些更高級的概念Send/Sync特徵(</mark><mark style="color:red;">實現此Trait的類型變數的引用可以安全在執行緒間共享。執行緒間轉移變數必須支援Send, 共享變數必須支援Sync)</mark><mark style="color:red;">，用來描述類型在併發環境下的特性</mark>。

`std::thread::spawn`函數就是一個普通函數，編譯器沒有對它做任何特殊處理。它能保證執行緒安全的關鍵是，它對參數有合理的約束條件。這樣的設計使得Rust在執行緒安全方面具備非常好的擴展性。很多高級的併發模型，在Rust中都可以通過協力廠商庫的形式實現，而且可以保證執行緒安全特性。只要對外API設計得合理，客戶就可以隨便使用這些並行庫，而不會有資料競爭的風險。

Rust的這個設計實際上將開發者分為了兩個陣營，一部分是核心庫的開發者，一部分是業務邏輯開發者。

* 對於一般的開發者來說，完全沒有必要寫任何unsafe程式碼，更沒有必要為自己的自訂類型去實現Sync Send，直接依賴編譯器的自動推導即可。這個陣營中的程式師可以完全享受編譯器和基礎庫帶來的安全性保證，無須投入太多精力去管理細節，極大地減輕腦力負擔。
* 而對於核心庫的開發者，則必須對Rust的這套機制非常瞭解。比如，他們可能需要設計自己的“無鎖資料類型”“執行緒池”“管道”等各種並行程式設計的基礎設施。這種時候，就有必要對這些類型使用unsafe implSend/Sync設計合適的介面。這些庫本身的內部實現是基於unsafe程式碼做的，它享受不到編譯器提供的各種安全檢查。

相反，這些庫本身才是保證業務邏輯程式碼安全性的關鍵。這個區分對整個生態圈是有好處的。跟前面講的“記憶體安全”問題類似，需要使用unsafe的程式碼總是很小的一部分，把這部分程式碼抽象到獨立的庫裡面，是比較容易測試和驗證它的正確性的。而業務邏輯程式碼才是千變萬化的，我們千萬不要在這部分程式碼中用unsafe做各種hack。因此，大部分普通人就能充分享受到少部分精英給我們提供的各種安全性保證（編譯器加基礎庫），放心大膽地做各種並行優化，完全不必擔心會製造執行緒安全問題。

## 執行緒(thread)

執行緒是作業系統能夠進行調度的最小單位，它是行程 (process)中的實際運作單位，每個滿程至少包含一個執行緒。

<mark style="color:red;">簡單的說，將所要執行的工作打包(通常寫成函數或閉包)，再將工作做為參數傳進執行緒中，待執行緒完成後傳回結果</mark>。

Rust 標準函式庫使用的是 <mark style="color:red;">1:1</mark> 的執行緒實作模型，也就是每一個語言產生的執行緒就是一個作業系統的執行緒。

在多核處理器越來越普及的今天，多執行緒程式設計也用得越來越廣泛。多執行緒的優勢有：

* 容易利用多核優勢；
* 比單執行緒反應更敏捷，比多進程資源分享更容易。

多執行緒程式設計在許多領域是不可或缺的。但是，多執行緒並行，非常容易引發資料競爭(同時有多個執行緒寫入同一區塊時發生)，而且還非常不容易被發現和除錯。

<mark style="color:red;">Rust標準庫中與執行緒相關的內容在</mark>[<mark style="color:red;">std::thread</mark>](https://doc.rust-lang.org/std/thread/index.html)<mark style="color:red;">模組中。Rust中的執行緒是對作業系統執行緒的直接封裝</mark>。

### spawn 建立新的執行緒，join 等待所有執行緒完成

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // 此範例是在1個thread中產生1~10的迴圈
    thread::spawn(|| {
        for i in 1..10 {
            println!("數字 {} 出現在產生的執行緒中！", i);
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 1..5 {
        println!("數字 {} 出現在主執行緒中！", i);
        thread::sleep(Duration::from_millis(1));
    }
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

/*
數字 1 出現在主執行緒中！
數字 1 出現在產生的執行緒中！
數字 2 出現在主執行緒中！
數字 2 出現在產生的執行緒中！
數字 3 出現在主執行緒中！
數字 3 出現在產生的執行緒中！
數字 4 出現在主執行緒中！
數字 4 出現在產生的執行緒中！
數字 5 出現在產生的執行緒中！
數字 6 出現在產生的執行緒中！
數字 7 出現在產生的執行緒中！
數字 8 出現在產生的執行緒中！
數字 9 出現在產生的執行緒中！
*/

// 以下範例是產生10個threads
use std::thread;
use std::time::Duration;

fn main() {
    let mut handles = Vec::new();
    for i in 1..10 {
        handles.push(thread::spawn(move || {
            let tid = thread::current().id();
            println!("TID: {:?}, 數字 {i} 出現在產生的執行緒中！", tid);
            thread::sleep(Duration::from_millis(5));
        }));
    }

    for i in 1..5 {
        println!("數字 {} 出現在主執行緒中！", i);
        thread::sleep(Duration::from_millis(1));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

```rust
use std::thread;
use std::time::Duration;
fn main() {
    // 使用Builder模式為child thread指定更多的訊息
    let t = thread::Builder::new()
        .name("child1".to_string())
        .spawn(
            // 新建的thread
            move || {
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

##
