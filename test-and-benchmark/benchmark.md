# 性能測試(benchmark)

## 簡介

單元測試是用來校驗程式的正確性的，而程式能正常運行後，往往還需要測試程式（一部分）的執行速度，這時就需要用到效能測試。&#x20;

通常來講，所謂效能測試，指的是測量程式執行的速度，即運行一次要多少時間（通常是執行多次求平均值）。

```rust
//src/lib.rs 檔案

#![feature(test)]
extern crate test;
pub fn add_two(a: i32) -> i32 {
    a + 2
}
#[cfg(test)]
mod tests {
    use super::*;
    use test::Bencher;
    #[test]
    fn it_works() {
        assert_eq!(4, add_two(2));
    }
    #[bench]
    fn bench_add_two(b: &mut Bencher) {
        b.iter(|| add_two(2));
    }
}
```
