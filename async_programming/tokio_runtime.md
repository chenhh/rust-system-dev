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

要使用tokio，需要先建立它提供的非同步執行時環境(Runtime)，然後在這個Runtime中執行非同步任務。(包含了類似python asyncio的事件迴圈，但功能更多)。

tokio提供了兩種工作模式的執行環境：

1. 單一執行緒的runtime(single thread runtime，也稱為current thread runtime) 。
2. 多執行緒(執行緒池)的runtime(multi thread runtime)。

這裡的所說的執行緒是Rust執行緒，而每一個Rust執行緒都是一個OS執行緒。

IO併發類任務較多時，單一線程的runtime性能不如多線程的runtime，但因為多線程runtime使用了多線程，使得線程間的通信變得更為複雜，也加重了線程間切換的開銷，使得有些情況下的性能不如使用單線程runtime。因此，在要求極限性能的時候，建議測試兩種工作模式的性能差距來選擇更優解。

### runtime的內部流程

<mark style="color:red;">非同步Runtime提供了非同步IO驅動、非同步計時器等非同步API，還提供了任務的排程器(scheduler)和Reactor事件迴圈(Event Loop)。</mark>每當建立一個Runtime時，就在這個Runtime中建立好了一個Reactor和一個Scheduler，同時還建立了一個任務佇列。&#x20;

從這一點看來，非同步執行時和作業系統的程序排程方式是類似的，只不過現代作業系統的程序排程邏輯要比非同步執行時的排程邏輯複雜的多。

當一個非同步任務需要執行，這個任務要被放入到可執行的任務佇列(就緒佇列)，然後等待被排程，當一個非同步任務需要阻塞時(對應那些在同步環境下會阻塞的操作)，它被放進阻塞佇列。

阻塞佇列中的每一個被阻塞的任務，都需要等待Reactor收到對應的事件通知(比如IO完成的通知、睡眠完成的通知等)來喚醒它。當該任務被喚醒後，它將被放入就緒佇列，等待排程器的排程。

就緒佇列中的每一個任務都是可執行的任務，可隨時被排程器排程選中。排程時會選擇哪一個任務，是排程器根據排程演算法去決定的。某個任務被排程選中後，排程器將分配一個執行緒去執行該任務。

大方向上來說，有兩種排程策略：<mark style="background-color:red;">搶佔式排程和協作式排程</mark>。

* 搶佔式排程策略，排程器會在合適的時候(排程規則決定什麼是合適的時候)強行切換當前正在執行的排程單元(例如程序、執行緒)，避免某個任務長時間霸佔CPU從而導致其它任務出現飢餓。
* 協作式排程策略則不會強行切斷當前正在執行的單元，只有執行單元執行完任務或主動放棄CPU，才會將該執行單元重新排隊等待下次排程，這可能會導致某個長時間計算的任務霸佔CPU，但是可以讓任務充分執行儘早完成，而不會被中斷。

對於面向大眾使用的作業系統(如Linux)通常採用搶佔式排程策略來保證系統安全，避免惡意程式霸佔CPU。而對於語言層面來說，通常採用協作式排程策略，這樣既有底層OS的搶佔式保底，又有協作式的高效。<mark style="color:red;">tokio的排程策略是協作式排程策略</mark>。

也可以簡單粗暴地去理解非同步排程：任務剛出生時，放進任務佇列尾部，排程器總是從任務佇列的頭部選擇任務去執行，執行任務時，如果任務有阻塞操作，則該任務總是會被放入到任務佇列的尾部。如果任務佇列的第一個任務都是阻塞的(即任務之前被阻塞但目前尚未完成)，則排程器直接將它重新放回佇列的尾部。因此，排程器總是從前向後一次又一次地輪詢這個任務佇列。當然，排程演算法通常會比這種簡單的方式要複雜的多，它可能會採用多個任務佇列，多種挑選標準，且佇列不是簡單的佇列，而是更高效的資料結構。

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

## spawn: 向runtime中添加新的非同步任務

有時候，定義要執行的非同步任務時，並未身處runtime內部。例如定義一個非同步函數，此時可以使用tokio::spawn()來生成非同步任務。

```rust
use std::thread;

use chrono::Local;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

// 在runtime外部定義一個異步任務，且該函數返回值不是Future類型
fn async_task() {
    println!("create an async task: {}", now());
    tokio::spawn(async {
        time::sleep(time::Duration::from_secs(2)).await;
        println!("async task over: {}", now());
    });
}

fn main() {
    let rt1 = Runtime::new().unwrap();
    rt1.block_on(async {
        // 調用函數，該函數內創建了一個異步任務，將在當前runtime內執行
        async_task();
    });
}

```

除了tokio::spawn()，runtime自身也能spawn，因此，也可以傳遞runtime(注意，要傳遞runtime的引用)，然後使用runtime的spawn()。

```rust
use tokio::{runtime::Runtime, time};

fn async_task(rt: &Runtime) {
    rt.spawn(async {
        println!("runtime spawn.");
        time::sleep(time::Duration::from_secs(10)).await;
    });
}

fn main() {
    let rt = Runtime::new().unwrap();
    rt.block_on(async {
        async_task(&rt);
    });
}
```

## 進入runtime: 非阻塞的enter()

`block_on()`是進入runtime的主要方式，會阻塞當前執行緒。

`enter()`進入runtime時，不會阻塞當前執行緒，它會返回一個EnterGuard。EnterGuard沒有其它作用，它僅僅只是宣告從它開始的所有非同步任務都將在runtime上下文中執行，直到刪除該EnterGuard。

刪除EnterGuard並不會刪除runtime，只是釋放之前的runtime上下文宣告。因此，刪除EnterGuard之後，可以宣告另一個EnterGuard，這可以再次進入runtime的上下文環境。

```rust
use chrono::Local;
use std::thread;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt = Runtime::new().unwrap();

    // 進入runtime，但不阻塞當前執行緒
    let guard1 = rt.enter();

    // 生成的非同步任務將放入當前的runtime上下文中執行
    tokio::spawn(async {
        time::sleep(time::Duration::from_secs(3)).await;
        println!("task1 sleep over: {}", now());
    });

    // 釋放runtime上下文，這並不會刪除runtime
    drop(guard1);

    // 可以再次進入runtime
    let guard2 = rt.enter();
    tokio::spawn(async {
        time::sleep(time::Duration::from_secs(1)).await;
        println!("task2 sleep over: {}", now());
    });

    drop(guard2);

    // 阻塞當前執行緒，等待非同步任務的完成
    // 必須等久一點，不然main thread會先完成
    thread::sleep(std::time::Duration::from_secs(4));
    println!("main thread complete.");
}

/*
task2 sleep over: 2024-10-20 02:50:57
task1 sleep over: 2024-10-20 02:50:59
main thread complete.
*/
```

## tokio的兩種執行緒：worker thread和blocking thread

tokio提供了兩種功能的執行緒：

* 用於非同步任務的工作執行緒(worker thread) 。
* 用於同步任務的阻塞執行緒(blocking thread)。

單個執行緒或多個執行緒的runtime，指的都是工作執行緒，即只用於執行非同步任務的執行緒，這些任務主要是IO密集型的任務。tokio預設會將每一個工作執行緒均勻地繫結到每一個CPU核心上。

但是，有些必要的任務可能會長時間計算而佔用執行緒，甚至任務可能是同步的，它會直接阻塞整個執行緒(比如thread::time::sleep())，這類任務如果計算時間或阻塞時間較短，勉強可以考慮留在非同步佇列中，但如果任務計算時間或阻塞時間可能會較長，它們將不適合放在非同步佇列中，因為它們會破壞非同步排程，使得同執行緒中的其它非同步任務處於長時間等待狀態，也就是說，這些非同步任務可能會被餓很長一段時間。

例如，直接在runtime中執行阻塞執行緒的操作，由於這類阻塞操作不在tokio系統內，tokio無法識別這類執行緒阻塞的操作，tokio只能等待該執行緒阻塞操作的結束，才能重新獲得那個執行緒的管理權。換句話說，worker thread被執行緒阻塞的時候，它已經脫離了tokio的控制，在一定程度上破壞了tokio的排程系統。

因此，tokio提供了這兩類不同的執行緒。worker thread只用於執行那些非同步任務，非同步任務指的是不會阻塞執行緒的任務。而一旦遇到本該阻塞但卻不會阻塞的操作(如使用tokio::time::sleep()而不是std::sleep())，會直接放棄CPU，將執行緒交還給排程器，使該執行緒能夠再次被排程器分配到其它非同步任務。blocking thread則用於那些長時間計算的或阻塞整個執行緒的任務。

<mark style="color:red;">blocking thread預設是不存在的，只有在呼叫了spawn\_blocking()時才會建立一個對應的blocking thread</mark>。

blocking thread不用於執行非同步任務，因此runtime不會去排程管理這類執行緒，它們在本質上相當於一個獨立的thread::spawn()建立的執行緒，它也不會像block\_on()一樣會阻塞當前執行緒。它和獨立執行緒的唯一區別，是blocking thread是在runtime內的，可以在runtime內對它們使用一些非同步操作，例如await。

```rust
use chrono::Local;
use std::thread;
use tokio::{self, runtime::Runtime, time};

fn now() -> String {
    Local::now().format("%F %T").to_string()
}

fn main() {
    let rt1 = Runtime::new().unwrap();
    // 建立一個blocking thread，可立即執行(由作業系統排程系統決定何時執行)
    // 注意，不阻塞當前執行緒
    let task = rt1.spawn_blocking(|| {
        println!("in task: {}", now());
        // 注意，是執行緒的睡眠，不是tokio的睡眠，因此會阻塞整個執行緒
        thread::sleep(std::time::Duration::from_secs(3))
    });

    // 小睡1毫秒，讓上面的blocking thread先執行起來
    std::thread::sleep(std::time::Duration::from_millis(1));
    println!("not blocking: {}", now());

    // 可在runtime內等待blocking_thread的完成
    rt1.block_on(async {
        task.await.unwrap();
        println!("after blocking task: {}", now());
    });
}
/*
not blocking: 2024-10-20 03:00:14
in task: 2024-10-20 03:00:14
after blocking task: 2024-10-20 03:00:17
*/

```

需注意，blocking thread生成的任務雖然繫結了runtime，但是它不是非同步任務，不受tokio排程系統控制。因此，如果在block\_on()中生成了blocking thread或普通的執行緒，block\_on()不會等待這些執行緒的完成。

```rust
rt.block_on(async{
  // 生成一個blocking thread和一個獨立的thread
  // block on不會阻塞等待兩個執行緒終止，因此block_on在這裡會立即返回
  rt.spawn_blocking(|| std::thread::sleep(std::time::Duration::from_secs(10)));
  thread::spawn(||std::thread::sleep(std::time::Duration::from_secs(10)));
});
```

tokio允許的blocking thread佇列很長(預設512個)，且可以在runtime build時通過max\_blocking\_threads()組態最大長度。如果超出了最大佇列長度，新的任務將放在一個等待佇列中進行等待(比如當前已經有512個正在執行的任務，下一個任務將等待，直到有某個blocking thread空閒)。

blocking thread執行完對應任務後，並不會立即釋放，而是繼續保持活動狀態一段時間，此時它們的狀態是空閒狀態。當空閒時長超出一定時間後(可在runtime build時通過thread\_keep\_alive()組態空閒的超時時長)，該空閒執行緒將被釋放。

blocking thread有時候是非常友好的，它像獨立執行緒一樣，但又和runtime繫結，它不受tokio的排程系統排程，tokio不會把其它任務放進該執行緒，也不會把該執行緒內的任務轉移到其它執行緒。換言之，它有機會完完整整地發揮單個執行緒的全部能力，而不像worker執行緒一樣，可能會被排程器打斷。

## 參考資料

* [https://shihyu.github.io/rust\_hacks/ch100/00.html](https://shihyu.github.io/rust\_hacks/ch100/00.html)
* [https://tokio.rs/blog/2019-10-scheduler](https://tokio.rs/blog/2019-10-scheduler)
