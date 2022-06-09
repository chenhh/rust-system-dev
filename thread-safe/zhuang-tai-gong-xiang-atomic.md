# 狀態共享atomic

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
