# 執行緒池

## threadpool

```rust
extern crate num_cpus;
extern crate threadpool;

use std::thread;
use std::time;
use threadpool::ThreadPool;
fn main() {
    let ncpus = num_cpus::get();
    let pool = ThreadPool::new(ncpus);
    for i in 0..ncpus * 5 {
        pool.execute(move || println!("this is thread number {}", i));
    }
    thread::sleep(time::Duration::from_millis(50));
    println!("finish of main function");
}
```

## thread local變數

對於多執行緒編程，執行緒區域性變量在一些場景下非常有用，而 Rust 通過標准庫和三方庫對此進行了支援。

使用標準庫的`thread_local` 巨集可以初始化執行緒區域性變量，然後在執行緒內部使用該變量的 `with` 方法獲取變量值：

```rust
use std::cell::RefCell;
use std::thread;

fn main() {
    thread_local!(static FOO: RefCell<u32> = RefCell::new(1));

    FOO.with(|f| {
        assert_eq!(*f.borrow(), 1);
        *f.borrow_mut() = 2;
    });

    // 每個執行緒開始時都會拿到執行緒區域性變量的FOO的初始值
    let t = thread::spawn(move || {
        FOO.with(|f| {
            assert_eq!(*f.borrow(), 1);
            *f.borrow_mut() = 3;
        });
    });

    // 等待執行緒完成
    t.join().unwrap();

    // 盡管子執行緒中修改為了3，我們在這裡依然擁有main執行緒中的區域性值：2
    FOO.with(|f| {
        assert_eq!(*f.borrow(), 2);
    });
}
```

##
