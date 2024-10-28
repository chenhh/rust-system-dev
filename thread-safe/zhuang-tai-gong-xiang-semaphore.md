# 狀態共享Semaphore

在多執行緒中，另一個重要的概念就是<mark style="color:red;">訊號量(semaphore)，使用它可以讓我們精準的控制當前正在執行的任務最大數量</mark>。

而在實際使用中，也有很多時候，我們需要通過訊號量來控制最大並發數，防止伺服器資源被撐爆。

本來 Rust 在標准庫中有提供一個訊號量實現，是由於各種原因這個庫現在已經不再推薦使用了，推薦使用tokio中提供的Semaphone實現: `tokio::sync::Semaphore`。

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();

    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            //
            // 在這裡執行任務...
            //
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```

上面程式碼建立了一個容量為 3 的訊號量，當正在執行的任務超過 3 時，剩下的任務需要等待正在執行任務完成並減少訊號量後到 3 以內時，才能繼續執行。

這裡的關鍵其實說白了就在於：訊號量的申請和歸還，使用前需要申請訊號量，如果容量滿了，就需要等待；使用後需要釋放訊號量，以便其它等待者可以繼續。
