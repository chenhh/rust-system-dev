# 日期與時間

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

