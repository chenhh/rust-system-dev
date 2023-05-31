# 運算子重載

## add(加法)

* [https://doc.rust-lang.org/std/ops/trait.Add.html](https://doc.rust-lang.org/std/ops/trait.Add.html)

```rust
use std::ops::Add;

#[derive(Debug, Copy, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Self;

    fn add(self, other: Self) -> Self {
        Self {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    let res = Point { x: 1, y: 0 } + 
        Point { x: 2, y: 3 } + 
        Point { x: 3, y: 3 }; // Point { x: 6, y: 6 }

    println!("{:?}", res);
}

```

