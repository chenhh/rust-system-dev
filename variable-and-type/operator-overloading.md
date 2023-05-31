# 運算子重載

## 算數運算子

算數運算子都有對應的trait的，他們都在[std::ops](https://doc.rust-lang.org/std/ops/index.html)下：

* \+: 加法。實現了std::ops::Add。&#x20;
* \-: 減法。實現了std::ops::Sub。&#x20;
* \*: 乘法。實現了std::ops::Mul。&#x20;
* /: 除法。實現了std::ops::Div。
* %: 取餘。實現了std::ops::Rem。

## 位元運算子

和算數運算子類似，位元運算也有對應的trait。

* &: 與操作。實現了std::ops::BitAnd。
* \|: 或操作。實現了std::ops::BitOr。
* ^: 異或。實現了std::ops::BitXor。
* &#x20;<<: 左移運算子。實現了std::ops::Shl。
* \>>: 右移運算子。實現了std::ops::Shr。

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

