# 常用字串操作

## 印出16進位

[fmt syntax](https://doc.rust-lang.org/std/fmt/index.html)。

[u32::from\_str\_radix](https://doc.rust-lang.org/std/primitive.u32.html#method.from\_str\_radix)。從str讀取數值，指定基數後，轉為u32類型之值。

```rust
use std::u32;

fn to_hex(val: &str, len: usize) -> String {
    let n: u32 = u32::from_str_radix(val, 2).unwrap();
    //  01$x essentially means that the number must be printed 
    // in hexadecimal form padded with zeros up to the length 
    // passed as the second (1st with zero-based indexing) argument.
    format!("{:01$x}", n, len * 2)
}

fn main() {
    println!("{}", to_hex("110011111111101010110010000100", 6))
    // prints 000033feac84
}
```

[https://blog.csdn.net/boysoft2002/article/details/131008787](https://blog.csdn.net/boysoft2002/article/details/131008787)
