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

* 這裡雖然使用了 `extern crate test`;，但是項目的 `Cargo.toml` 檔案中依賴區並不需要新增對 `test` 的依賴；
* 評測函數 `fn bench_add_two(b: &mut Bencher) {}` 上面使用 `#[bench]` 做標註，同時函數接受一個參數，`b` 就是 Rust 提供的評測器。這個寫法是固定的。

然後，在工程根目錄下，執行：`cargo bench`。即可進行性能測試。

Rust 的效能測試是以納秒 ns 為單位。

寫測評程式碼的時候，需要注意以下一些點：

1. 只把你需要做效能測試的程式碼（函數）放在評測函數中；&#x20;
2. 對於參與做效能測試的程式碼（函數），要求每次測試做同樣的事情，不要做累積和改變外部狀態的操作；
3. 參數效能測試的程式碼（函數），執行時間不要太長。太長的話，最好分成幾個部分測試。這也方便找出效能瓶頸所在地方。

