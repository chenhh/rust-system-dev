# 狀態共享condvar

## Condvar

Condvar是條件變數，它可以用於等待某個事件的發生。在等待的時候，這個執行緒處於阻塞狀態，並不消耗CPU資源。

在常見的作業系統上，Condvar的內部實現是調用的作業系統提供的條件變數。它調用wait方法的時候需要一個MutexGuard類型的參數，因此Condvar總是與Mutex配合使用的。而且我們一定要注意，一個Condvar應該總是對應一個Mutex，不可混用，否則會導致執行階段的panic。

Condvar的一個常見使用模式是和一個Mutex\<bool>類型結合使用。我們可以用Mutex中的bool變數存儲一個舊的狀態，在條件發生改變的時候修改它的狀態。通過這個狀態值，我們可以決定是否需要執行等待事件的操作。

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;
fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone();
    thread::spawn(move || {
        thread::sleep(Duration::from_secs(1));
        let &(ref lock, ref cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
        *started = true;
        cvar.notify_one();
        println!("child thread {}", *started);
    });
    // wait for the thread to start up
    let &(ref lock, ref cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    println!("before wait {}", *started);
    while !*started {
        started = cvar.wait(started).unwrap();
    }
    println!("after wait {}", *started);
}
```

這段程式碼中存在兩個執行緒之間的共用變數，包括一個Condvar和一個Mutex封裝起來的bool類型。我們用Arc類型把它們包起來。在子執行緒中，我們做完了某件工作之後，就將共用的bool類型變數設置為true，並使用Condvar：：notify\_one通知事件發生。在主執行緒中，我們首先判定這個bool變數是否為true：如果已經是true，那就沒必要進入等候狀態了；否則，就進入阻塞狀態，等待子執行緒完成任務。
