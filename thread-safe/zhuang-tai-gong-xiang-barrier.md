# 狀態共享Barrier

## Barrier

Barrier是這樣的一個類型，它使用一個整數做初始化，可以使得多個執行緒在某個點上一起等待，然後再繼續執行。

```rust
use std::sync::{Arc, Barrier};
use std::ops::Deref;
use std::thread;
fn main() {
    // barrier::new(n)之值為能夠擋住n-1個執行緒
    // 而在第n個thread呼叫wait()時一次釋放
    let barrier = Arc::new(Barrier::new(10));
    let mut handlers = vec![];
    for idx in 0..10 {
        // 設定c為共用的barrier
        let c = barrier.clone();
        let t = thread::spawn(move || {
            println!("{idx}: before wait");
            // 每一個thread在此處等待，因此所有的before wait都會先印出之後
            // 才會再印出after wait
            c.wait();   // auto deref, c.deref().wait();
            println!("{idx}: after wait");
        });
        handlers.push(t);
    }
    for h in handlers {
        h.join().ok();
    }
}
```

這個程式創建了一個多個執行緒之間共用的Barrier，它的初始值是10，表示最多擋9個執行緒，第10個執行緒呼叫`wait()`後會一次釋放。

我們創建了10個子執行緒，每個子執行緒都有一個`Arc`指標指向了這個`Barrier`，並在子執行緒中調用了`Barrier::wait`方法。這些子執行緒執行到`wait`方法的時候，就開始進入等候狀態，一直到wait方法被調用了10次，10個子執行緒都進入等候狀態，此時Barrier就通知這些執行緒可以繼續了。然後它們再開始執行下面的邏輯。所以最終的執行結果是：先列印出10條before wait，再列印出10條after wait，絕不會錯亂。
