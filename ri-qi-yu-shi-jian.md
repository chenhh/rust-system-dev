# 日期與時間

## [Module std::time](https://doc.rust-lang.org/stable/std/time/index.html)

\[[中文](https://rustwiki.org/zh-CN/std/time/index.html)]

* struct Duration：類型代表時間跨度，通常用於系統超時。&#x20;
* struct Instant： 單調非遞減時鐘的度量。 不透明且僅對 Duration 有用。

```rust
use std::thread::sleep;
use std::time::{Duration, Instant};

fn expensive_function() {
    // 睡2秒
    sleep(Duration::new(2, 0));
}

fn main() {
    let start = Instant::now();
    expensive_function();
    let duration = start.elapsed();

    println!("Time elapsed in expensive_function() is: {:?}", duration);
}
```

## [chrono crate](https://docs.rs/chrono/latest/chrono/)

```rust
use chrono::prelude::*;

fn main() {
    // UTC時區 // 2022-06-01T14:46:30.033807237Z
    let utc: DateTime<Utc> = Utc::now(); 
    // 本地時區, // 2022-06-01T14:46:30.033810947+00:00
    let local: DateTime<Local> = Local::now(); 

    println!("{:?}", utc);
    println!("{:?}", local);
}
```

