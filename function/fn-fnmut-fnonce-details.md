# Fn, FnMut, FnOnce的區別

## 簡介

在Rust裡，閉包被分為了三種類型，列舉如下

* [Fn(\&self)](https://doc.rust-lang.org/std/ops/trait.Fn.html)。
* [FnMut(\&mut self)](https://doc.rust-lang.org/std/ops/trait.FnMut.html)。
* [FnOnce(self)](https://doc.rust-lang.org/std/ops/trait.FnOnce.html)。

## Fn

```rust
pub trait Fn<Args>: FnMut<Args>
where
    Args: Tuple,
{
    // Required method
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}
```



## 參考資料

* [https://dengjianping.github.io/2019/03/05/%E8%B0%88%E4%B8%80%E8%B0%88Fn,-FnMut,-FnOnce%E7%9A%84%E5%8C%BA%E5%88%AB.html](https://dengjianping.github.io/2019/03/05/%E8%B0%88%E4%B8%80%E8%B0%88Fn,-FnMut,-FnOnce%E7%9A%84%E5%8C%BA%E5%88%AB.html)
