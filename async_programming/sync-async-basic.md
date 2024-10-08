# 同步與非同步基礎

非同步程式解決等待的問題。

一般的程式，其實大部分都在處理等待問題，像是 GUI 程式，使用者按了一個按鈕後，不可能完全卡在那邊等待其他的程式跑完，又像是API server 經由網路呼叫一個第三方的外部程式，如果在那邊等待，是不是浪費了很多 CPU資源。

這時可以直接用執行緒操作，避免主程式被卡住的情況，而其實執行緒也算是一種非同步程式的模型，而且現今的程式語言也有更好語法去使用它們，像是用 Future or Promise。

在瞭解完要解決什麼問題後，需要知道的是不同解法之間的差異。如python在非同步流程控制上比較多重點在 callback, eventloop, coroutine 還有最後衍生出來的 async/await。

但取而代之的就是，程式會有點難寫難讀，所以就有了 libev, libevent, libuv 這類函式庫幫忙處理 非同步IO 這部分，使用這類的函式庫，上層程式很簡單就可以使用回調(callback)函數與之互動，等到 socket 的 file descriptor 被觸發時再去呼叫回調函數繼續處理下去。

很快就有人發現可以寫出 callback hell 這種程式，非同步程式 (主要談callback) 到這邊就變成語法的改進了，希望能夠把程式寫得更漂亮更易讀一點，所以就有了 promise 或是協程(coroutine)的方法與其結合，在協程中會把控制權從函數中切換回主程式中，某種程度跟 eventloop 就很相似，所以可以利用協程的特性加上非阻塞I/O 成為更好的框架，而程式也會變得像是同步程式，不過要注意一但有程式是阻塞IO 或是CPU密集的任務，就會把這個執行緒卡住，最後提到的 async/await 只不過是語法的變形，其實跟協程的概念是很相似的，可以讓整個程式更好寫易讀，然後背後又能高效率的處理 IO密集問題。

如果非同步程式只是處理網路IO 的話，應該是不會有太大的問題。但如果有個CPU密集的任務，最好還是要能生成行程/執行緒去處理，或是利用佇列丟給其它的程式去處理。所以一旦楚這些架構後，才能知道採取哪種方式處理問題是比較好的。

## 名詞解釋

### **Sync**（同步）：一件事情做完之後，再做下一件事情。

* **blocking**（堵塞）：指「等一件事情」的行為。

### **Async**（非同步）：一件事情還沒完成，可以做其他不衝突的事情。

* **concurrency**（並行、併發）：程式 **架構** 中，各個任務可以 **獨立執行** 的特性。
  * **future**：Rust 中的一個非同步任務的表示。
  * **polling**：不停地詢問任務，確認事情是否已經完成。
  * **event-driven**：事情完成後，任務自己發通知表明完成。

### **parallelism**（平行）：同時 **執行** 數個程式的行為。

* **thread**（執行緒、執行緒）：系統處理程式（任務集）的基本單元。執行緒通常是交由 CPU 核心執行。
  * **spawn**（生成）：指產生執行緒的行為。
  * **thread pool**（執行緒池）：將執行緒高效分配給每個任務的地方。

### **Async runtime**: 以 tokio 為例

* **join** (macro)：並行執行 async 函式，並在全部完成後回傳。
* **select** (macro)：哪個 async 函式快，回傳那個 async 函式的結果。
* **main** (attribute macro)：在 main() 初始化 runtime。
* **block\_on**：在 sync 上執行 async 函式。
* **spawn**：平行執行 async 函式。
* **spawn\_blocking**：在非同步函式裡面，為一個高耗時且同步 (blocking) 的函式另闢新執行緒 (thread)。

## 同步 (Synchronous) 跟非同步 (Asynchoronous)

「同步」就是整個程式等一件事情完成（blocking，堵塞）。「非同步」則是一件事情還沒完成，可以做其他不衝突的事情。

就以早餐來說，「同步」就像是等吐司烤完之後，才去準備花生醬（每件任務循序漸進地執行）；「非同步」就像是吐司正在烤的時候，就先準備花生醬（每件任務並行地執行）。

<figure><img src="../.gitbook/assets/image.png" alt="" width="563"><figcaption><p>同步與非同步任務</p></figcaption></figure>

## 並行 (Concurrency) 與平行 (Parallelism) <a href="#bing-xing-concurrency-yu-ping-xing-parallelism" id="bing-xing-concurrency-yu-ping-xing-parallelism"></a>

**並行** 就是指程式架構，任務可以(前後)互不相關執行的特性。我們可以稱「吐司正在烤」和「準備花生醬」是個獨立任務，而我們可以 **並行** 地進行這些任務。

假如你媽也想幫你一起做早餐，而你負責「烤土司、準備醬料」，而你媽負責「煎蛋、擺盤」。你做自己的任務、而你媽做自己的任務——這就是 **平行**。

。我們可以把「你」和「你媽」比擬為 **CPU 核心 (core)** ，分配給你和你媽的一大堆獨立任務叫做 **執行緒 (thread)** 。**並行** 就是工作單元自己用非同步的方式處理任務；**平行** 就是分配其他工作單元處理任務。

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>平行與併行</p></figcaption></figure>

## 執行緒池 (thread pool)

電腦的核心是有限的。那要怎麼高效的把一大堆執行緒，都分配到這些 CPU 上呢？通常把這個「分配」叫做排程 (scheduling，排程）。首先，最簡單的做法就是自己跟系統開執行緒（通常我們把這種執行緒叫做 OS thread）。

<figure><img src="../.gitbook/assets/image (2).png" alt="" width="563"><figcaption><p>執行緒池</p></figcaption></figure>

```rust
// OS thread in rust
let handle = std::thread::spawn(|| {
  /* 你的同步作業 */
});
```

這樣的好處是不需要在程式裡面包一個排程器，但缺點是執行緒的 **spawn** 生成（把任務分配給CPU）和 **destroy** 銷毀（告訴CPU不用繼續執行任務）都得找你的作業系統操作。

那如果我們可以自己排程，是不是就能減少找作業系統的開銷，甚至是做到更多更高效的事情（比如在一個執行緒裡面執行數個並行任務）？

首先，我們需要先跟作業系統說「我需要 N 個可以分配任務的執行緒，」然後把這些執行緒都放進去我們的執行緒池。接下來程式需要執行緒執行任務，就請執行緒池分配。而我們從池子分配到的執行緒，就是 green thread。

但假如每個執行緒裡面都是會堵住程式的任務 (blocking)，那 T執行緒池裡面的 OS Thread 就得等這些任務完成，最後反而沒有達到預期的高效分配。<mark style="color:red;">因此，如果 threads 裡面是完全 sync 的任務，就</mark> <mark style="color:red;"></mark><mark style="color:red;">**沒有必要用上 thread pool**</mark><mark style="color:red;">，讓 OS 分配即可</mark>。

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption><p>執行緒池</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (4).png" alt="" width="563"><figcaption><p>執行緒池內均為同步任務</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (5).png" alt="" width="563"><figcaption><p>執行緒池內有并發任務</p></figcaption></figure>

## Rust的Future 與 Executor <a href="#jin-jie-yue-du-future-yu-executor" id="jin-jie-yue-du-future-yu-executor"></a>

怎麼讓 Rust 知道一個任務是否完成？

Rust 的 Future 就是一個可並行任務的抽象表示。而 Executor 就是負責輪詢 (polling) Futures 的程式。

1. Executor 會從 Future Receiver 抓出一個 Future。
2. Executor 執行這個任務的 `poll()`，並觀察其回應。
3. 如果是 `Poll::Pending`（處理中），就繼續處理下個任務。
4. 如果是 `Poll::Ready(T)`（已完成，回傳值是 T），則將回傳值交給需求方。

<figure><img src="../.gitbook/assets/image (6).png" alt="" width="563"><figcaption><p>Future任務流程</p></figcaption></figure>

但這個文字流程有個問題：poll() 只會執行一次嗎？如果會執行數次，那 poll() 下次會在什麼時候執行？這裡就得提到 Rust 的 waker 機制了。每一次 poll()，Executor 都會給這個任務一個 context。裡面有一個 waker，可以用來提醒 Executor「可以執行 poll() 了。」

倘若如果我們可以等到作業完成，再執行 wake() 呢？要這麼做，我們就得先知道「工作什麼時候才完成？」如果任務是用 callback 或 event 告知任務狀態的，那就是在收到 event、或 callback 觸發進行呼叫。

## async 函式、區塊和 await

Rust 中 Future 與 Executor 的理論基礎，但實務上沒有這麼麻煩。事實上在 Rust 中，用 async 函式和 block 是非常直覺的。

```rust
// 異步版本
sync fn make_breakfast() -> Toast {
  let toast = bake_toast().await;
  let butter = prepare_peanut_butter().await;

  toast.apply(butter);
  toast
}

// 同步版本
fn make_breakfast() -> Toast {
  let toast = bake_toast();
  let butter = prepare_peanut_butter();

  toast.apply(butter);
  toast
}
```

雖然整體上「做早餐」還是循序執行的（先烤完吐司、才準備花生醬），但做早餐這件事情因為已經是非同步的了，所以你可以在做早餐的時候做其他事情。

可發現到 .await 剛好就是「任務切換點」。.await 之後，你可以去做其他事情（而不是空等）。等到烤箱聲音響了之後 (wake()) 之後再回來做剩下的事情。

所以整體上 async 函式是比較高效的，但我們要怎麼讓整個任務更高效呢（在 async 裡面一次性執行更多工？）

<figure><img src="../.gitbook/assets/image (8).png" alt="" width="375"><figcaption><p>非同步版本</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (10).png" alt="" width="375"><figcaption><p>同步版本</p></figcaption></figure>

## 在 async 函式裡面並行執行數個任務 (futures)

希望在一個 async 裡面一次性執行數個任務。這裡我們可以借助 tokio 的 join!() 工具巨集，表示「我希望這兩個任務同時操作」，就像是把這兩個任務融合為一了。

```rust
async fn make_breakfast() -> Toast {
  let (toast, butter) = tokio::join!(
    // 要注意這裡不需要 .await，
    // await 的事情 `join!()` 會處理。
    bake_toast(),
    prepare_peanut_butter()
  );

  toast.apply(butter);
  toast
}
```

<figure><img src="../.gitbook/assets/image (11).png" alt="" width="563"><figcaption><p>併發處理任務</p></figcaption></figure>

換一種現實中也常遇到的例子：你希望早餐可以在小孩上學前做完，如果沒做完就不要繼續做了。所以我們想要設定一個計時器，如果計時到了還沒做完就直接取消；反之就繼續做：

<figure><img src="../.gitbook/assets/image (12).png" alt="" width="563"><figcaption><p>定時任務并行</p></figcaption></figure>

可以用 tokio::select!()——同時等「做早餐」跟「計時器」，回傳完成速度最快的任務（分支），而取消剩下沒做完的任務（分支）。

```rust
// Option 包含「有」或「沒有」兩種可能。如果計時器到了，
// 吐司還沒完成，那就沒有早餐；反之，就有早餐。
async fn make_breakfast_with_timer() -> Option<Toast> {
  tokio::select! {
    // 如果早餐先完成，那就有早餐。
    toast = make_breakfast() => Some(toast),

    // 如果時間先到，那就沒早餐。
    _ = timer() => None,
  }
}

/// 一個設定在 30 分鐘的計時器
async fn timer() {
  tokio::time::sleep(
    std::time::Duration::from_secs(30 /* min */ * 60 /* sec */)
  ).await
}

async fn make_breakfast() -> Toast {
  let (toast, butter) = tokio::join!(
    bake_toast(),
    prepare_peanut_butter()
  );

  toast.apply(butter);
  toast
}
```

## await 只能在 async function 裡面執行

是 .await 只能在 async block 或 async function 裡面使用。也就是說，你不能在同步函式（包括 main()）裡面呼叫非同步函式：

```rust
fn main() {
  // 會編譯錯誤！
  let file_content = make_breakfast().await;

  // 還是不行 😄
  let file_content = async {
    make_breakfast().await
  }.await; /* async block 也需要 await */
}

```

既然每個呼叫者都必須是 async 的，那是誰呼叫第一個 async 函式呢？ 這就得提到 Async Runtime 了。

## 從 Future 看 async 和 await

async fn 其實展開來看，就是一個回傳 Future 的函式：

```rust
struct ReadFileFuture { ... }

impl Future for ReadFileFuture {
  type Output = String;

  fn poll(...) { ... }
}

fn read_file(path: &Path) -> ReadFileFuture {}
```

而 await 的大致意思就是「沒完成就說整個函式沒完成；完成就繼續」：

```rust
// 把這個函式的 context 轉交給 read_to_string
let content_status = tokio::fs::read_to_string(path).poll(cx);

let content = match content_status {
  // 如果這個 feature 沒完成，就剩下的 async 也就無法繼續。
  Poll::Pending => return Poll::Pending,

  // 反之，把值拿回來
  Poll::Ready(c) => c,
}
```

實際上這部分還有許多地方需要考慮：包括要怎麼在下次呼叫 poll() 的時候，知道現在要繼續執行哪個 Future。

## Rust 的 async runtime

實務上你不需要自己寫一個 executor，而是使用現成的 async runtime（執行階段、執行時）。一個 async runtime 除了 executor 之外，還有提供很多功能（比如上文提及的 thread pool、工具巨集和函式，以及檔案讀寫、channel 等等常用功能的非同步對應方法）。

常見的 async runtime 有 tokio、async-std 和 smol，其中又以 tokio 和 async-std 為大宗。

## 讓 main() 變成 async 函式的起源地

那 main() 原則上就是組態 runtime，讓 runtime 準備 executor 的地方。

```rust
// 手動設定
fn main() {
  // 設定多執行緒的 tokio runtime。
  tokio::runtime::Builder::new_multi_thread()
    // 啟用所有功能。
    .enable_all()
    // 建構 runtime。
    .build()
    // 如果 runtime 建構失敗就停住整個程式。
    .unwrap()
    // async 函式的起源地——堵塞 (blocking)，
    // 讓整個 main() 等待這個 async 函式完成。
    .block_on(async {
        println!("Hello world");
    })

  // 然後程式就可以結束了。
}

// 巨集設定
#[tokio::main]
async fn main() {
  println!("Hello, World!");
}
```

## 讓一個任務 (Future) 變成一個綠色執行緒 (Green Thread)——spawn

然大部分的情況下，在 單執行緒「並行」就已經很足夠快了。倘若這個任務耗時很長，你希望開另一條執行緒「平行」專門處理這個任務，那就可以用 spawn：

```rust
let handle = tokio::task::spawn(async {
  /* 現在這裡面的東西，都在獨立的 thread 裡面跑了！ */
});
```

[`tokio::task::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html) 雖然用起來很像建立 OS thread 的 `std::thread::spawn`，但 **spawn 裡面不要放高耗時的同步函式**——除非你樂見整個 runtime 被卡在一件任務上面（或者是直接把 runtime 搞死，直接 panic！）

那要怎麼在非同步函式裡面，開另一個 thread 跑同步函式呢？你可以用接下來會提到的 `tokio::task::spawn_blocking`。

## 在非同步函式裡面呼叫高耗時同步函式——spawn\_blocking

除了開一個 `std::thread::spawn` OS thread 跑這種函式之外，你也可以用\
[`tokio::task::spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn\_blocking.html) 開一個 **可以 await** 的同步 _blocking_ 堵塞函式。

```rust
let _this_returns_42 = tokio::task::spawn_blocking(|| {
  for i in 0..114514 {
    for j in 0..1919810 {}
  }

  42
}).await;
```

這樣子跑高耗時的函式之時，照樣可以執行其他不用堵塞的任務。同理，你也可以把這個套進去 `join!` 並行完成，可是 **這樣建立出的 thread 是取消不了的——不只是單純的 `select!`，還包含 `.abort()`** 。因此還是盡量選擇並善用非同步函式。



## 參考資料

* [https://blog.pan93.com/what-is-rust-async/](https://blog.pan93.com/what-is-rust-async/)
* [https://shihyu.github.io/rust\_hacks/ch100/00.html](https://shihyu.github.io/rust\_hacks/ch100/00.html)\

* [Tokio學習筆記](https://skyao.io/learning-tokio/docs/introduction.html)\
