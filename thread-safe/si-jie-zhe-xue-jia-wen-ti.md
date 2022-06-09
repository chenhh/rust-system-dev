# 死結-哲學家問題

## Deadlock

假設有5個哲學家，共用一張放有5把椅子的桌子，每人分得一把椅子，但是，桌子上共有5支筷子，在每人兩邊各放一支，哲學家們在肚子饑餓時才試圖分兩次從兩邊拿起筷子就餐。條件:·拿到兩支筷子時哲學家才開始吃飯；·如果筷子已在他人手上，則該哲學家必須等他人吃完之後才能拿到筷子；·任一哲學家在自己未拿到兩隻筷子前卻不放下自己手中的筷子。

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;
struct Philosopher {
    name: String,
    left: usize,
    right: usize,
}
impl Philosopher {
    fn new(name: &str, left: usize, right: usize) -> Philosopher {
        Philosopher {
            name: name.to_string(),
            left: left,
            right: right,
        }
    }
    fn eat(&self, table: &Table) {
        let _left = table.forks[self.left].lock().unwrap();
        println!("{} take left fork.", self.name);
        thread::sleep(Duration::from_secs(2));
        let _right = table.forks[self.right].lock().unwrap();
        println!("{} take right fork.", self.name);
        thread::sleep(Duration::from_secs(1));
        println!("{} is done eating.", self.name);
    }
}
struct Table {
    forks: Vec<Mutex<()>>,
}
fn main() {
    let table = Arc::new(Table {
        forks: vec![
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
            Mutex::new(()),
        ],
    });
    let philosophers = vec![
        Philosopher::new("Judith Butler", 0, 1),
        Philosopher::new("Gilles Deleuze", 1, 2),
        Philosopher::new("Karl Marx", 2, 3),
        Philosopher::new("Emma Goldman", 3, 4),
        Philosopher::new("Michel Foucault", 4, 0),
    ];
    let handles: Vec<_> = philosophers
        .into_iter()
        .map(|p| {
            let table = table.clone();
            thread::spawn(move || {
                p.eat(&table);
            })
        })
        .collect();
    for h in handles {
        h.join().unwrap();
    }
}
```

我們可以發現，5個哲學家都拿到了他左邊的那支筷子，而都在等待他右邊的那支筷子。在沒等到右邊筷子的時候，每個人都不會釋放自己已經拿到的那支筷子。於是，大家都進入了無限的等待之中，程式無法繼續執行了。這就是“鎖死”。在Rust中，“鎖死”問題是沒有辦法在編譯階段由靜態檢查來解決的。就像前面提到的“迴圈引用製造記憶體洩漏”一樣，編譯器無法通過靜態檢查來完全避免這個問題，需要程式設計師自己注意。
