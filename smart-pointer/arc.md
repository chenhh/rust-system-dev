# Arc

## 簡介

Arc 是 Atomic Rc 的縮寫，顧名思義：原子化的 Rc 智慧指標。

原子化或者其它鎖雖然可以帶來的執行緒安全，但是都會伴隨著效能損耗，而且這種效能損耗還不小。因此 Rust 把這種選擇權交給你，畢竟需要執行緒安全的程式碼其實佔比並不高，大部分時候我們開發的程式都在一個執行緒內，此時使用Rc即可。

Arc 和 Rc 擁有完全一樣的 API，但Arc 和 Rc 並沒有定義在同一個模組，前者通過 use std::sync::Arc 來引入，後者通過 use std::rc::Rc。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let s = Arc::new(String::from("mutli-thread msg"));
    for _ in 0..10 {
        let s = Arc::clone(&s);
        let handle = thread::spawn(move || {
           println!("tid:{:?}, {}", thread::current().id(), s)
        });
        // no-join
    }
}
```

