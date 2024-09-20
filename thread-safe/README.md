---
description: 線程
---

# 執行緒安全

## 簡介

Rust不僅在沒有自動垃圾回收（Garbage Collection）的條件下實現了記憶體安全，而且實現了執行緒安全。Rust編譯器可以在編譯階段避免所有的資料競爭（Data Race）問題。這也是Rust的核心競爭力之一。

<mark style="color:red;">Rust語言本身並不知曉“執行緒”“併發”具體是什麼，而是抽象出了一些更高級的概念Send/Sync，用來描述類型在併發環境下的特性</mark>。`std::thread::spawn`函數就是一個普通函數，編譯器沒有對它做任何特殊處理。它能保證執行緒安全的關鍵是，它對參數有合理的約束條件。這樣的設計使得Rust在執行緒安全方面具備非常好的擴展性。很多高級的併發模型，在Rust中都可以通過協力廠商庫的形式實現，而且可以保證執行緒安全特性。只要對外API設計得合理，客戶就可以隨便使用這些並行庫，而不會有資料競爭的風險。

Rust的這個設計實際上將開發者分為了兩個陣營，一部分是核心庫的開發者，一部分是業務邏輯開發者。

* 對於一般的業務開發者來說，完全沒有必要寫任何unsafe程式碼，更沒有必要為自己的自訂類型去實現Sync Send，直接依賴編譯器的自動推導即可。這個陣營中的程式師可以完全享受編譯器和基礎庫帶來的安全性保證，無須投入太多精力去管理細節，極大地減輕腦力負擔。
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

###

##

## 避免資料競爭(race condition)

```rust
use std::thread;
fn main() {
    let mut health = 12;
    // spawn函數接受的參數是一個閉包。
    // 我們在閉包裡面引用了函數體內的區域變數，
    // 而這個閉包是執行在另外一個執行緒上，
    // 編譯器無法肯定區域變數health的生命週期一定大於閉包的生命週期，
    // 於是發生了錯誤。
    // 解法：move
    thread::spawn(move || {
        health *= 2;
    });
    // 12, 值並未改變
    // ？因為health是Copy類型，在遇到move型閉包的時候，
    // 閉包內的health實際上是一份新的複製，外面的變數並沒有被真正修改。
    println!("{}", health);  // 12
}
```

那我們究竟怎樣做才能讓一個變數在不同執行緒中共用呢？<mark style="background-color:red;">**答案是：我們沒有辦法在多執行緒中直接讀寫普通的共用變數，除非使用Rust提供的執行緒安全相關的設施**</mark>。

也就是說，Rust給我們提供了一個重要的安全保證：The compiler prevents all data races。**資料競爭，意思是在多執行緒程式中，不同執行緒在沒有使用同步的條件下並行訪問同一塊資料，且其中至少有一個是寫操作的情況**。

在許多傳統（非函數式）的程式設計語言中，並行程式設計是非常困難的，原因就在於程式碼中存在大量的共用狀態和很多隱藏的資料依賴。程式設計師必須非常清楚程式碼的流程，使用合適的策略正確實現併發控制。而萬一某人在某個地方犯了一個小錯誤，那麼這個程式就成了不安全的，而且沒有什麼靜態檢查工具可以保證完整無遺漏地將此類問題檢查出來。

根據資料競爭的定義，它的發生需要三個條件：

* <mark style="color:red;">資料共用</mark>：有多個執行緒同時訪問一份資料；
* <mark style="color:red;">資料修改</mark>：至少存在一個執行緒對資料做修改；
* <mark style="color:red;">沒有同步</mark>：至少存在一個執行緒對資料的訪問沒有使用同步措施。

<mark style="background-color:red;">**我們只要讓這三個條件無法同時發生即可**</mark><mark style="background-color:red;">。</mark>

* 可以禁止資料共用，比如actor-based concurrency，多執行緒之間的通信僅靠發送消息來實現，而不是通過共用資料來實現；
* 可以禁止資料修改，比如functional programming，許多函數式程式設計語言嚴格限制了資料的可變性，而對共用性沒有限制。

然而以上設計在許多情況下都有一些性能上的缺陷，無法達到“零開銷抽象”的目的。

傳統C/C++需要依賴程式師不犯錯誤來保證執行緒安全；而Rust是由工具自動保證的。

## Send, Sync trait簡介

Rust執行緒安全背後的功臣是兩個特殊的trait。

* [std::marker::Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html)：如果類型`T`實現了`Sync`類型，那說明在不同的執行緒中使用`&T`訪問同一個變數是安全的。
* [std::marker::Send](https://doc.rust-lang.org/std/marker/trait.Send.html)：如果類型`T`實現了`Send`類型，那說明這個類型的變數在不同的執行緒中傳遞所有權是安全的。

Rust把類型根據`Sync`和`Send`做了分類。不可能隨意在多執行緒之間共用變數，也不可能在使用多執行緒共用的時候忘記加鎖。除非你使用unsafe，否則不可能寫出存在“資料競爭”的程式碼來。

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where F: FnOnce() -> T, F: Send + 'static, T: Send + 'static
```

參數類型`F`有重要的約束條件`F：Send+'static`，`T：Send+'static`。但凡在執行緒之間傳遞所有權會發生安全問題的類型，都無法在這個參數中出現，否則就是編譯錯誤

。另外，Rust對全域變數也有很多限制，你不可能簡單地通過全域變數在多執行緒中隨意共用狀態。這樣，編譯器就會禁止掉可能有危險的執行緒間共用資料的行為。

在Rust中，執行緒安全是預設行為，大部分類型在單執行緒中是可以隨意共用的，但是沒辦法直接在多執行緒中共用。也就是說，只要程式師不濫用unsafe，Rust編譯器就可以檢查出所有具有“資料競爭”潛在風險的程式碼。凡<mark style="color:red;">是通過了編譯檢查的程式碼，Rust可以保證，絕對不會出現“執行緒不安全”的行為</mark>。如此一來，多執行緒程式碼和單執行緒程式碼就有了嚴格的分野。一般情況下，我們不需要考慮多執行緒的問題。即便是萬一不小心在多執行緒中訪問了原本只設計為單執行緒使用的程式碼，編譯器也會報錯。

## Send trait

<mark style="color:red;">根據定義：如果類型T實現了Send trait，那說明這個類型的變數在不同執行緒中傳遞所有權是安全的</mark>。究竟具備什麼特點的類型才滿足Send約束？

<mark style="background-color:red;">**如果一個類型可以安全地從一個執行緒move進入另一個執行緒，那它就是Send類型**</mark>。比如：普通的數字類型是`Send`，因為我們把數字move進入另一個執行緒之後，兩個執行緒同時執行也不會造成什麼安全問題。

更進一步，<mark style="color:red;background-color:red;">**內部不包含引用的類型，都是Send**</mark>。因為這樣的類型跟外界沒有什麼關聯，當它被move進入另一個執行緒之後，它所有的部分都跟原來的執行緒沒什麼關係了，不會出現併發訪問的情況。比如String類型。

<mark style="background-color:red;">**具有泛型參數的類型，是否滿足Send大多是取決於參數類型是否滿足Send**</mark>。比如`Vec<T>`，只要我們能保證`T：Send`，那麼`Vec<T>`肯定也是`Send`，把它move進入其他執行緒是沒什麼問題的。再比如`Cell<T>`、`RefCell<T>`、`Option<T>`、`Box<T>`，也都是這種情況。

還有一些類型，不論泛型參數是否滿足Send，都是滿足Send的。<mark style="color:red;">這種類型，可以看作一種“構造器”，把不滿足Send條件的類型用它包起來，就變成了滿足Send條件的類型</mark>。比如`Mutex<T>`就是這種。`Mutex<T>`這個類型實際上不關心它內部類型是怎樣的，反正要訪問內部資料，一定要調用`lock()`方法上鎖，它的所有權在哪個執行緒中並不重要，所以把它move到其他執行緒也是沒有問題的。

那麼什麼樣的類型不是`Send`呢？典型的如`Rc<T>`類型。我們知道，Rc是引用計數指標，把Rc類型的變數move進入另外一個執行緒，只是其中一個引用計數指標move到了其他執行緒，這樣會導致不同的執行緒中的Rc變數引用同一塊資料，Rc內部實現沒有做任何執行緒同步處理，這是肯定有問題的。所以標準庫中早已指定Rc不是Send。當我們試圖在執行緒邊界傳遞這個類型的時候，就會出現編譯錯誤。

但是相對的是，`Arc<T>`類型是符合`Send`的（當然需要`T：Send`）。為什麼呢？因為Arc類型內部的引用計數用的是“原子計數”，對它進行增減操作，不會出現多執行緒資料競爭。所以，多個執行緒擁有指向同一個變數的Arc指標是可以接受的。

## sync trait

<mark style="color:red;">Sync的定義是，如果類型</mark><mark style="color:red;">`T`</mark><mark style="color:red;">實現了</mark><mark style="color:red;">`Sync`</mark> <mark style="color:red;">trait，那說明在不同的執行緒中使用</mark><mark style="color:red;">`&T`</mark><mark style="color:red;">（唯讀）訪問同一個變數是安全的</mark>。

顯然，基本數字類型肯定是`Sync`。假如不同執行緒都擁有指向同一個`i32`類型的唯讀引用`&i32`，這是沒什麼問題的。因為這個類型引用只能讀，不能寫。多個執行緒讀同一個整數是安全的。

大部分具有泛型參數的類型是否滿足`Sync`，很多都是取決於參數類型是否滿足`Sync`。像`Box<T>`、`Vec<T>、Option<T>`這種也是`Sync`的，只要其中的參數`T`是滿足`Sync`的。

也有一些類型，不論泛型參數是否滿足`Sync`，它都是滿足`Sync`的。這種類型把不滿足`Sync`條件的類型用它包起來，就變成了滿足`Sync`條件的。e就是這種。多個執行緒同時擁有`&Mutex<T>`型引用，指向同一個變數是沒問題的。

那麼什麼樣的類型是！Sync呢？<mark style="color:red;">**所有具有“內部可變性”而又沒有多執行緒同步考慮的類型都不是Sync的。**</mark>比如，`Cell<T>`和`RefCell<T>`就不能是`Sync`的。按照定義，如果我們多個執行緒中都持有指向同一個變數的`&Cell<T>`型指標，那麼在多個執行緒中，都可以執行`Cell::set`方法來修改它裡面的資料。而我們知道，這個方法在修改內部資料的時候，是沒有考慮多執行緒同步問題的。所以，我們必須把它標記為！Sync。

還有一些特殊的類型，它們既具備內部可變性，又滿足`Sync`約束，比如前面提到的`Mutex<T>`類型。為什麼說`Mutex<T>`具備內部可變性？大家查一下文檔就會知道，這個類型可以通過不可變引用調用`lock()`方法，返回一個智慧指標`MutexGuard<T>`類型，而這個智慧指標有權修改內部資料。這個做法就跟`RefCell<T>`的`try_borrow_mut()`方法非常類似。區別只是`Mutex::lock()`方法的實現，使用了作業系統提供的多執行緒同步機制，實現了執行緒同步，保證了非同步安全；而`RefCell`的內部實現就是簡單的普通數字加減操作。因此，Mutex\<T>既具備內部可變性，又滿足Sync約束。<mark style="background-color:red;">除了Mutex\<T>，標準庫中還有RwLock\<T>、AtomicBool、AtomicIsize、AtomicUsize、AtomicPtr等類型，都提供了內部可變性，而且滿足Sync約束</mark>。

## 自動推理

`Send`和`Sync`是marker trait。在Rust中，<mark style="background-color:blue;">有一些trait是在</mark><mark style="background-color:blue;">`std::marker`</mark><mark style="background-color:blue;">模組中的特殊trait。它們有一個共同的特點，就是內部都沒有任何的方法，它們只用於給類型做“標記”。</mark>在std::marker這個模組中的trait，都是給類型做標記的trait。每一種標記都將類型嚴格切分成了兩個組。

```rust
unsafe impl Send for .. { }
unsafe impl Sync for .. { }
```

這是一個臨時的、特殊的語法，<mark style="background-color:red;">**它的含義是：針對所有類型，預設實現了Send/Sync**</mark><mark style="background-color:red;">。使用了這種特殊語法的trait叫作OIBIT（Opt-in builtintrait），後來改稱為Auto Trait。</mark>注意：這個語法是不穩定的，以後會改變。不管怎樣，編譯器留了一個後門，可以讓我們定義Auto Trait。

Auto Trait有一個重要特點，就是編譯器允許用戶不用手寫impl，自動根據這個類型的成員“推理”出這個類型是否滿足這個trait。我們可以手動指定這個類型滿足這個trait約束，也可以手動指定它不滿足這個trait約束，但是手動指定的時候，一定要用unsafe關鍵字。

<mark style="background-color:blue;">使用！Send這種寫法表示“取反”操作，這些類型就一定不滿足Send約束</mark>。請大家一定要注意unsafe關鍵字。這個關鍵字在這裡的意思是，編譯器自己並沒有能力正確地、智慧地理解每一個類型的內部實現原理，並由此判斷它是否滿足Send或者Sync。它需要程式設計師來提供這個資訊。

## 參考資料

* [\[知乎\] 多線程編程的秘密：Sync, Send, and 'Static](https://zhuanlan.zhihu.com/p/362285521)
