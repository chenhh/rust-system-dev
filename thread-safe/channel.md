# 管道(channel)

## 簡介

Rust標準庫中還提供了另外一種執行緒之間的通信方式:mpsc。存儲在`std::sync::mpsc`這個模組中。

<mark style="background-color:blue;">mpsc代表的是Multi-producer，single consumer FIFO queue，即多生產者單消費者先進先出佇列(queue)</mark>。這種執行緒之間的通信方式是在不同執行緒之間建立一個通信“管道”（channel），一邊發送消息，一邊接收消息，完成資訊交流。

## 異步管道

非同步管道是最常用的一種管道類型。它的特點是:發送端和接收端之間存在一個緩衝區，發送端發送資料的時候，是先將這個資料扔到緩衝區，再由接收端自己去取。因此，每次發送，立馬就返回了，發送端不用管資料什麼時候被接收端處理。

```rust
use std::sync::mpsc::channel;
use std::thread;
fn main() {
    // 建立channel
    let (tx, rx) = channel();
    thread::spawn(move || {
        for i in 0..10 {
            tx.send(i).unwrap();
        }
    });
    while let Ok(r) = rx.recv() {
        println!("received {}", r);
    }
}
```

子執行緒中的發送者不斷迴圈調用send方法，發送資料。在主執行緒中，我們使用接收者不斷調用recv方法接收資料。我們可以注意到，channel（）是一個泛型函數，Sender和Receiver都是泛型類型，且一組發送者和接收者必定是同樣的類型參數，因此保證了發送和接收端都是同樣的類型。因為Rust中的類型推導功能的存在，使我們可以在調用channel的時候不指定具體類型參數，而通過後續的方法調用，推導出正確的類型參數。

Sender和Receiver的泛型參數必須滿足T：Send約束。這個條件是顯而易見的：被發送的消息會從一個執行緒轉移到另外一個執行緒，這個約束是為了滿足執行緒安全。如果用戶指定的泛型參數沒有滿足條件，在編譯的時候會發生錯誤，提醒我們修復bug。

發送者調用send方法，接收者調用recv方法，返回類型都是Result類型，用於錯誤處理，因為它們都有可能調用失敗。當發送者已經被銷毀的時候，接收者調用recv則會返回錯誤；同樣，當接收者已經銷毀的時候，發送者調用send也會返回錯誤。

在管道的接收端，如果調用recv方法的時候還沒有資料，它會進入等候狀態阻塞當前執行緒，直到接收到資料才繼續往下執行。管道還可以是多發送端單接收端。做法很簡單，只需將發送端Sender複製多份即可。複製方式是調用Sender類型的clone()方法。這個庫不支持多接收端的設計，因此Receiver類型沒有clone()方法。在上例的基礎上我們稍做改動，創建多個執行緒，每個執行緒發送一個資料到接收端。

```rust
use std::sync::mpsc::channel;
use std::thread;
fn main() {
    let (tx, rx) = channel();
    for i in 0..10 {
        // 複製一個新的 tx,將這個複製的變數 move 進入子執行緒
        let tx = tx.clone();
        thread::spawn(move || {
            tx.send(i).unwrap();
        });
    }
    drop(tx);
    while let Ok(r) = rx.recv() {
        println!("received {}", r);
    }
}
```

但在這個示例中，這些數字呈亂序排列，因為它們來自不同的執行緒，哪個先執行哪個後執行並不是確定的，取決於作業系統的調度。

## 同步管道

非同步管道內部有一個不限長度的緩衝區，可以一直往裡面填充資料，直至記憶體資源耗盡。非同步管道的發送端調用send方法不會發生阻塞，只要把消息加入到緩衝區，它就馬上返回。

同步管道的特點是：其內部有一個固定大小的緩衝區，用來緩存消息。如果緩衝區被填滿了，繼續調用send方法的時候會發生阻塞，等待接收端把緩衝區內的消息拿走才能繼續發送。緩衝區的長度可以在建立管道的時候設置，而且0是有效數值。如果緩衝區的長度設置為0，那就意味著每次的發送操作都會進入等候狀態，直到這個消息被接收端取走才能返回。

```rust
use std::sync::mpsc::sync_channel;
use std::thread;
fn main() {
    let (tx, rx) = sync_channel(1);
    tx.send(1).unwrap();
    println!("send first");
    thread::spawn(move || {
        tx.send(2).unwrap();
        println!("send second");
    });
    println!("receive first {}", rx.recv().unwrap());
    println!("receive second {}", rx.recv().unwrap());
}
/*
send first
receive first 1
receive second 2
*/
```

程式執行結果永遠是：發送一個並接收一個之後，才會出現發送第二個接收第二個。我們講的這兩種管道都是單向通信，一個發送一個接收，不能反過來。Rust沒有在標準庫中實現管道雙向通信。雙向管道也不是不可能的，在第三方庫中已經有了實現。
