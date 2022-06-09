# 狀態共享Barrier

## Barrier

Barrier是這樣的一個類型，它使用一個整數做初始化，可以使得多個執行緒在某個點上一起等待，然後再繼續執行。

```rust
use std::sync::{Arc, Barrier};
use std::thread;
fn main() {
    let barrier = Arc::new(Barrier::new(10));
    let mut handlers = vec![];
    for _ in 0..10 {
        let c = barrier.clone();
        // The same messages will be printed together.
        // You will NOT see any interleaving.
        let t = thread::spawn(move || {
            println!("before wait");
            c.wait();
            println!("after wait");
        });
        handlers.push(t);
    }
    for h in handlers {
        h.join().ok();
    }
}
```

這個程式創建了一個多個執行緒之間共用的Barrier，它的初始值是10。我們創建了10個子執行緒，每個子執行緒都有一個Arc指標指向了這個Barrier，並在子執行緒中調用了Barrier：：wait方法。這些子執行緒執行到wait方法的時候，就開始進入等候狀態，一直到wait方法被調用了10次，10個子執行緒都進入等候狀態，此時Barrier就通知這些執行緒可以繼續了。然後它們再開始執行下面的邏輯。所以最終的執行結果是：先列印出10條before wait，再列印出10條after wait，絕不會錯亂。
