# tokio執行環境

## 簡介

在使用tokio之前，應當先理解tokio的核心概念：runtime和task。

```toml
// cargo add tokio
// 開啟全部功能的tokio，
// 在瞭解tokio之後，只開啟需要的特性，減少編譯時間，減小編譯大小
tokio = {version = "1.4", features = ["full"]}
```

## 建立tokio Runtime

要使用tokio，需要先建立它提供的非同步執行時環境(Runtime)，然後在這個Runtime中執行非同步任務。(類似python asyncio的事件迴圈)。

tokio提供了兩種工作模式的執行環境：

1. 單一執行緒的runtime(single thread runtime，也稱為current thread runtime) 。
2. 多執行緒(執行緒池)的runtime(multi thread runtime)。

這裡的所說的執行緒是Rust執行緒，而每一個Rust執行緒都是一個OS執行緒。

IO併發類任務較多時，單一線程的runtime性能不如多線程的runtime，但因為多線程runtime使用了多線程，使得線程間的通信變得更為複雜，也加重了線程間切換的開銷，使得有些情況下的性能不如使用單線程runtime。因此，在要求極限性能的時候，建議測試兩種工作模式的性能差距來選擇更優解。

#### 使用tokio::runtime建立Runtime(使用預設參數)

```rust
use tokio;

fn main() {
  // 建立runtime，預設為多執行緒(與CPU核數相同)
  let rt = tokio::runtime::Runtime::new().unwrap();
  // 可將程式暫停10秒，去console使用ps -eLf | grep 'targe[t]查看執行緒數量
   std::thread::sleep(std::time::Duration::from_secs(10));
  
  // 只有明確指定，才能創建出單一線程的runtime
  let rt_s = tokio::runtime::Builder::new_current_thread().build().unwrap();
}
```

runtime結構定義如下：

```rust
#[derive(Debug)]
pub struct Runtime {
    /// Task scheduler
    scheduler: Scheduler,

    /// Handle to runtime, also contains driver handles
    handle: Handle,

    /// Blocking pool handle, used to signal shutdown
    blocking_pool: BlockingPool,
}
```

#### 也可以使用Runtime Builder來組態並建立runtime(可自訂參數)

```rust
use tokio;

fn main() {
  // 建立帶有執行緒池的runtime
  let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(8)  // 8個工作執行緒
    .enable_io()        // 可在runtime中使用非同步IO
    .enable_time()      // 可在runtime中使用非同步計時器(timer)
    .build()            // 建立runtime
    .unwrap();
}
```

## async main

對於main函數，tokio提供了簡化的非同步執行時創建方式。

通過#\[tokio::main]註解(annotation)，使得async main自身成為一個async runtime。

```rust
use tokio;

// 創建多線程runtime
#[tokio::main]
async fn main() {}


// 等價的創建法
#[tokio::main(flavor = "multi_thread"] // 等價於#[tokio::main]
#[tokio::main(flavor = "multi_thread", worker_threads = 10))]
#[tokio::main(worker_threads = 10))]
```

上述宣告等價於如下沒有使用`#[tokio::main]`的程式碼：

```rust
fn main(){
  tokio::runtime::Builder::new_multi_thread()
        .worker_threads(N)  
        .enable_all()
        .build()
        .unwrap()
        .block_on(async { // our program });
}
```

創建單一執行緒的main runtime

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() {}


// 等價於
fn main() {
    tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async { // our program })
}
```

## 多個runtime共存

可手動創建執行緒，並在不同執行緒內創建互相獨立的runtime。

```rust
use std::thread;
use std::time::Duration;
use tokio::runtime::Runtime;

fn main() {
    // 在第一個執行緒內創建一個多執行緒的runtime
    let t1 = thread::spawn(|| {
        let rt = Runtime::new().unwrap();
        thread::sleep(Duration::from_secs(10));
    });

    // 在第二個執行緒內創建一個多執行緒的runtime
    let t2 = thread::spawn(|| {
        let rt = Runtime::new().unwrap();
        thread::sleep(Duration::from_secs(10));
    });

    t1.join().unwrap();
    t2.join().unwrap();
}
```

## 在非同步runtime中執行非同步任務

多數時候，非同步任務是一些帶有網絡IO操作的任務。而在介紹tokio用法時，只需使用tokio的非同步計時器即可解釋清楚，如tokio::time::sleep()。

std::time也提供了sleep()，但它會阻塞整個執行緒，而tokio::time中的sleep()則只是讓它所在的任務放棄CPU並進入調度隊列等待被喚醒，它不會阻塞任何執行緒，它所在的執行緒仍然可被用來執行其它非同步任務。

```rust
use chrono::Local;
use tokio::runtime::Runtime;

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        println!("before sleep: {}", Local::now().format("%F %T.%3f"));
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("after sleep: {}", Local::now().format("%F %T.%3f"));
    });
}

// 等價程式碼
#[tokio::main]
async fn main() {
   
    println!("before sleep: {}", Local::now().format("%F %T.%3f"));
     // 只是定義了Future，此時尚未執行
    let task = tokio::time::sleep(tokio::time::Duration::from_secs(2));
     // ...
     // 開始執行task任務，並等待它執行完成
     task.await;
    println!("after sleep: {}", Local::now().format("%F %T.%3f"));
}

/*
before sleep: 2024-10-19 09:11:35.620
after sleep: 2024-10-19 09:11:37.622
*/

```

block\_on函數(會阻塞目前執行緒，是等待異步任務完成，而不是等待runtime中的所有任務都完成)要求一個Future物件作為參數，可以像上面一樣直接使用一個async {}來定義一個Future。每一個Future都是一個已經定義好但尚未執行的非同步任務，每一個非同步任務中可能會包含其它子任務。

這些非同步任務不會直接執行，需要先將它們放入到runtime環境，然後在合適的地方通過Future的await來執行它們。await可以將已經定義好的非同步任務立即加入到runtime的任務隊列中等待調度執行，於此同時，await會等待該非同步任務完成才返回。

block\_on也有返回值，其返回值為其所執行異步任務的返回值。

```rust
use tokio::{runtime::Runtime, time};

fn main() {
    let rt = Runtime::new().unwrap();
    let res: i32 = rt.block_on(async {
        time::sleep(time::Duration::from_secs(2)).await;
        3
    });
    println!("{}", res); // 3
}
```



## 參考資料

* [https://shihyu.github.io/rust\_hacks/ch100/00.html](https://shihyu.github.io/rust\_hacks/ch100/00.html)
