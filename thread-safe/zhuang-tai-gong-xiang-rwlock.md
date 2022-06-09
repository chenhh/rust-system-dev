# 狀態共享RWLock

## RwLock

RwLock就是“讀寫鎖”。它跟Mutex很像，主要區別是對外暴露的API不一樣。對Mutex內部的資料讀寫，RwLock都是調用同樣的lock方法；而對RwLock內部的資料讀寫，它分別提供了一個成員方法read/write來做這個事情。其他方面基本和Mutex一致。

```rust
use std::sync::Arc;
use std::sync::RwLock;
use std::thread;
const COUNT: u32 = 1000000;
fn main() {
    let global = Arc::new(RwLock::new(0));
    let clone1 = global.clone();
    let thread1 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone1.write().unwrap();
            *value += 1;
        }
    });
    let clone2 = global.clone();
    let thread2 = thread::spawn(move || {
        for _ in 0..COUNT {
            let mut value = clone2.write().unwrap();
            *value -= 1;
        }
    });
    thread1.join().ok();
    thread2.join().ok();
    println!("final value: {:?}", global);
}
```

##
