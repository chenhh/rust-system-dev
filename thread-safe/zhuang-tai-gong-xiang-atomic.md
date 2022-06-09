# 狀態共享atomic

## 簡介

Mutex用起來簡單，但是無法並發讀，RwLock可以並發讀，但是使用場景較為受限且效能不夠。

從 Rust1.34 版本後，就正式支援原子型別。<mark style="color:red;">原子指的是一系列不可被 CPU 上下文交換的機器指令，這些指令組合在一起就形成了原子操作</mark>。在多核 CPU 下，當某個 CPU 核心開始執行原子操作時，會先暫停其它 CPU 核心對記憶體的操作，以保證原子操作不會被其它 CPU 核心所干擾。

由於原子操作是通過指令提供的支援，因此它的效能相比鎖和訊息傳遞會好很多。相比較於鎖而言，原子型別不需要開發者處理加鎖和釋放鎖的問題，同時支援修改，讀取等操作，還具備較高的並發效能，幾乎所有的語言都支援原子型別。

可以看出原子型別是無鎖型別，但是無鎖不代表無需等待，因為原子型別內部使用了CAS(Compare and swap)迴圈，當大量的沖突發生時，該等待還是得等待！但是總歸比鎖要好。

CAS 全稱是 Compare and swap, 它通過一條指令讀取指定的記憶體位址，然後判斷其中的值是否等於給定的前置值，如果相等，則將其修改為新的值。

## Atomic

Rust標準庫還為我們提供了一系列的“原子操作”資料類型，它們在std::sync::atomic模組裡面。它們都是符合Sync的，可以在多執行緒之間共用。比如，我們有AtomicIsize類型，顧名思義，它對應的是isize類型的“執行緒安全”版本。我們知道，普通的整數讀取再寫入，這種操作是非原子的。而原子整數的特點是，可以把“讀取”“計算”“再寫入”這樣的操作編譯為特殊的CPU指令，保證這個過程是原子操作。

```rust
use std::sync::atomic::{AtomicIsize, Ordering};
use std::thread;
use std::sync::Arc;
const COUNT: u32 = 1000000;
fn main() {
    // Atomic 系列類型同樣提供了執行緒安全版本的內部可變性
    let global = Arc::new(AtomicIsize::new(0));
    let clone1 = global.clone();
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            clone1.fetch_add(1, Ordering::SeqCst);
        }
    });
    let clone2 = global.clone();
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            clone2.fetch_sub(1, Ordering::SeqCst);
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```
