# tokio

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

#### 使用tokio::runtime建立Runtime(使用預設參數)

```rust
use tokio;

fn main() {
  // 建立runtime
  let rt: Runtime = tokio::runtime::Runtime::new().unwrap();
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



## 參考資料

* [https://shihyu.github.io/rust\_hacks/ch100/00.html](https://shihyu.github.io/rust\_hacks/ch100/00.html)
