# 狀態共享RWLock

## RwLock

Mutex會對每次讀寫都進行加鎖，但某些時候，我們需要大量的並發讀，Mutex就無法滿足需求了，此時就可以使用RwLock。

這種類型的鎖在任何時間點允許多個讀取器或最多一個寫入器。此鎖的寫入部分通常允許修改底層數據（獨佔訪問），而此鎖的讀取部分通常允許只讀訪問（共享訪問）。

<mark style="color:red;">相比之下，Mutex不區分獲取鎖的讀取器或寫入器，因此阻塞任何等待鎖可用的線程。RwLock只要寫入者不持有鎖，將允許任意數量的讀取者獲取鎖</mark>。

一個RwLock, 像Mutex, 會因恐慌而中毒。但是請注意，RwLock只有在獨佔鎖定（寫入模式）時發生恐慌時，才可能會中毒。如果任何讀者發生恐慌，那麼鎖就不會中毒。

```rust
use std::sync::RwLock;

fn main() {
    let lock = RwLock::new(5);

    // 同一時間允許多個讀
    {
        let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } // 讀鎖在此處被drop

    // 同一時間只允許一個寫
    {
        let mut w = lock.write().unwrap();
        *w += 1;
        assert_eq!(*w, 6);

        // 以下程式碼會panic，因為讀和寫不允許同時存在
        // 寫鎖w直到該語句塊結束才被釋放，因此下面的讀鎖依然處於`w`的作用域中
        // let r1 = lock.read();
        // println!("{:?}",r1);
    } // 寫鎖在此處被drop
}
```

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

## Mutex 還是 RwLock&#x20;

首先簡單性上Mutex完勝，因為使用RwLock你得操心幾個問題：

* 讀和寫不能同時發生，如果使用try\_xxx解決，就必須做大量的錯誤處理和失敗重試機制。
* 當讀多寫少時，寫操作可能會因為一直無法獲得鎖導致連續多次失敗(writer starvation)。
* RwLock 其實是作業系統提供的，實現原理要比Mutex復雜的多，因此單就鎖的效能而言，比不上原生實現的Mutex。

簡單總結下兩者的使用場景：

* 追求高並發讀取時，使用RwLock，因為Mutex一次只允許一個執行緒去讀取。
* 如果要保證寫操作的成功性，使用Mutex。
* 不知道哪個合適，統一使用Mutex。

需要注意的是，RwLock雖然看上去貌似提供了高並發讀取的能力，但這個不能說明它的效能比Mutex高，<mark style="background-color:red;">事實上Mutex效能要好不少，後者唯一的問題也僅僅在於不能並發讀取</mark>。

總之，如果你要使用RwLock要確保滿足以下兩個條件：並發讀，且需要對讀到的資源進行"長時間"的操作。
