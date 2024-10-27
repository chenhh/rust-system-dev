# 管道(channel)

## 簡介

Rust標準庫中還提供了另外一種執行緒之間的(阻塞)通信方式:mpsc。存儲在[`std::sync::mpsc`](https://rustwiki.org/zh-CN/std/sync/mpsc/index.html)這個模組中。

<mark style="background-color:blue;">mpsc代表的是Multi-producer，single consumer FIFO queue，即多生產者單消費者先進先出佇列(queue)</mark>。這種執行緒之間的通信方式是在不同執行緒之間建立一個通信“管道”（channel），一邊發送消息(傳送者, transmitter)，一邊接收消息(接收者, receiver)，完成資訊交流。

程式碼中的一部分會呼叫傳送者的方法來傳送你想要傳遞的資料，然後另一部分的程式碼會檢查接收者收到的訊息。當傳送者或接收者有一方被釋放掉時，該通道就會被關閉。

該模塊通過通道提供基於訊息的通訊，具體定義為以下三種類型：

* Sender
* SyncSender
* Receiver

Sender 或 SyncSender 用於將資料發送到 Receiver。 兩個發送者都是可克隆的 (multi-producer)，因此許多執行緒可以同時發送到一個接收者 (single-consumer)。

這些通道有兩種類型：

* <mark style="color:red;">異步，無限緩沖的通道</mark>。 channel 函數將返回 (Sender, Receiver) 元組，其中所有發送都是異步的 (它們從不阻塞)。 通道在概念上具有無限的緩沖區。
* <mark style="color:red;">同步的有界通道</mark>。 sync\_channel 函數將返回 (SyncSender, Receiver) 元組，其中待處理訊息的存儲是固定大小的預分配緩沖區。 所有的發送都將被阻塞，直到有可用的緩沖區空間。 請注意，允許的界限為 0，從而使通道成為 “rendezvous” 通道，每個發送者在該通道上原子地將訊息傳遞給接收者。

![channel模型](../.gitbook/assets/thread\_channel-min.PNG)

## 異步管道

非同步管道是最常用的一種管道類型。<mark style="color:red;">它的特點是:發送端和接收端之間存在一個緩衝區，發送端發送資料的時候，是先將這個資料扔到緩衝區，再由接收端自己去取。</mark>因此，每次發送，立刻就返回了，發送端不用管資料什麼時候被接收端處理。

```rust
use std::sync::mpsc::channel;
use std::sync::mpsc::{Sender, Receiver};
use std::thread;
use std::time;

fn main() {
    // 建立channel, 發送與接受端必須為同類型，但可由編譯器推論決定
    // 有多種類型需要傳送時，可建立多個channel
    let (tx, rx): (Sender<_>, Receiver<_>) = channel();
    
    let interval = time::Duration::from_millis(20);
    
    // tx已經move給thread，無法再被使用
    let handle = thread::spawn(move || {
        for i in 0..10 {
            tx.send(i).unwrap();
            thread::sleep(interval);
        }
    });

    while let Ok(r) = rx.recv() {
        println!("received {}", r);
    }
    handle.join();
}
```

子執行緒中的發送者不斷迴圈調用`send`方法，發送資料。

在主執行緒中，我們使用接收者不斷調用`recv`方法接收資料。我們可以注意到，`channel()`是一個泛型函數，Sender和Receiver都是泛型類型，且一組發送者和接收者必定是同樣的類型參數，因此保證了發送和接收端都是同樣的類型。因為Rust中的類型推導功能的存在，使我們可以在調用channel的時候不指定具體類型參數，而通過後續的方法調用，推導出正確的類型參數。

Sender和Receiver的泛型參數必須滿足`T：Send`約束。這個條件是顯而易見的：<mark style="color:red;">被發送的消息會從一個執行緒轉移到另外一個執行緒，這個約束是為了滿足執行緒安全。</mark>如果用戶指定的泛型參數沒有滿足條件，在編譯的時候會發生錯誤，提醒我們修復bug。

發送者調用send方法，接收者調用recv方法，返回類型都是Result類型，用於錯誤處理，因為它們都有可能調用失敗。當發送者已經被銷毀的時候，接收者調用recv則會返回錯誤；同樣，當接收者已經銷毀的時候，發送者調用send也會返回錯誤。

在管道的接收端，如果調用recv方法的時候還沒有資料，它會進入等候狀態阻塞當前執行緒，直到接收到資料才繼續往下執行。管道還可以是多發送端單接收端。做法很簡單，只需將發送端Sender複製多份即可。複製方式是調用Sender類型的clone()方法。這個庫不支持多接收端的設計，因此Receiver類型沒有clone()方法。在上例的基礎上我們稍做改動，創建多個執行緒，每個執行緒發送一個資料到接收端。

```rust
use std::sync::mpsc::channel;
use std::thread;
fn main() {
    let (tx, rx) = channel();
    for _ in 0..2 {
        // 複製一個新的 tx,將這個複製的變數 move 進入子執行緒
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..5 {
                tx.send(i).unwrap();
            }
        });
    }
    // drop(tx);
    while let Ok(r) = rx.recv() {
        println!("received {r}");
    }
}
/*
received 0
received 1
received 2
received 3
received 4
received 0
received 1
received 2
received 3
received 4
...
*/
```

但在這個示例中，這些數字呈亂序排列，因為它們來自不同的執行緒，哪個先執行哪個後執行並不是確定的，取決於作業系統的調度。

### 使用管道傳送struct

```rust
use std::sync::mpsc::channel;
use std::thread;
struct Block {
    value: i32,
}
fn main() {
    let (tx1, rx1) = channel::<Block>();
    let (tx2, rx2) = channel::<Block>();
    // receiver 1 => send 2
    thread::spawn(move || {
        let mut block = rx1.recv().unwrap();
        println!("Input: {:?}", block.value); // 1
        block.value += 1;
        tx2.send(block).unwrap();
    });
    let input = Block { value: 1 };
    // send 1 => receiver 1
    tx1.send(input).unwrap();
    // send 2 => receiver 2
    let output = rx2.recv().unwrap();
    println!("Output: {:?}", output.value); // 2
}
```

### 使用管道傳送引用

```rust
use std::sync::mpsc;
use std::thread;
trait Message: Send {
    fn print(&self);
}
struct Msg1 {
    value: i32,
}
impl Message for Msg1 {
    fn print(&self) {
        println!("value: {:?}", self.value);
    }
}
fn main() {
    let (tx, rx) = mpsc::channel::<Box<dyn Message>>();
    let handle = thread::spawn(move || {
        let msg = rx.recv().unwrap();
        msg.print();    // 1
    });
    let msg = Box::new(Msg1 { value: 1 });
    tx.send(msg).unwrap();
    handle.join().ok();
}
```

### 傳輸多種類型的資料

如果你想要傳輸多種類型的數據，可以為每個類型創建一個通道，你也可以使用枚舉類型來實現。

枚舉類型還能讓我們帶上想要傳輸的資料，但是有一點需要注意，Rust 會按照枚舉中佔用記憶體最大的那個成員進行記憶體對齊，這意味著就算你傳輸的是枚舉中佔用記憶體最小的成員，它佔用的記憶體依然和最大的成員相同, 因此會造成記憶體上的浪費。

```rust
use std::sync::mpsc::{self, Receiver, Sender};

enum Fruit {
    Apple(u8),
    Orange(String),
}

fn main() {
    let (tx, rx): (Sender<Fruit>, Receiver<Fruit>) = mpsc::channel();

    tx.send(Fruit::Orange("sweet".to_string())).unwrap();
    tx.send(Fruit::Apple(2)).unwrap();

    for _ in 0..2 {
        match rx.recv().unwrap() {
            Fruit::Apple(count) => println!("received {} apples", count),
            Fruit::Orange(flavor) => println!("received {} oranges", flavor),
        }
    }
}

/*
received sweet oranges
received 2 apples
*/
```

## 同步管道

<mark style="color:red;">非同步管道內部有一個不限長度的緩衝區，可以一直往裡面填充資料，直至記憶體資源耗盡。</mark>非同步管道的發送端調用send方法不會發生阻塞，只要把消息加入到緩衝區，它就馬上返回。

<mark style="color:red;">同步管道的特點是：其內部有一個固定大小的緩衝區，用來緩存消息。如果緩衝區被填滿了，繼續調用send方法的時候會發生阻塞，等待接收端把緩衝區內的消息拿走才能繼續發送。</mark>緩衝區的長度可以在建立管道的時候設置，而且0是有效數值。如果緩衝區的長度設置為0，那就意味著每次的發送操作都會進入等候狀態，直到這個消息被接收端取走才能返回。

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
