# 同步與非同步

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

## 參考資料

[https://blog.pan93.com/what-is-rust-async/](https://blog.pan93.com/what-is-rust-async/)\
