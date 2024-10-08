# 第三方套件

## [awesome rust (github)](https://github.com/rust-unofficial/awesome-rust)

* [\[知乎\] GitHub 上有哪些值得關注的 Rust 項目？](https://www.zhihu.com/question/30511494/answer/649921526)
* [rust-boom](https://github.com/rust-boom/rust-boom)

## 數值計算

* [num-bigint](https://crates.io/crates/num-bigint)：官方的大整數計算。
* ndarray：
  * [Benchmarking Rust's Ndarray Crate Against Plain Slices](https://www.reidatcheson.com/rust/ndarray/performance/2022/06/11/rust-ndarray.html)
  *
* ndarray-rand：
* ndarray-stats：
* [polars](https://github.com/pola-rs/polars)：快速多線程的DataFrame函式庫。
* [noisy\_float](https://docs.rs/noisy\_float/0.2.0/noisy\_float/)：這個板塊包含了浮點類型，如果它們被設置為非法值，例如NaN，就會發生panic!。
* [stars](https://github.com/statrs-dev/statrs)：統計套件。
* [glam-rs](https://github.com/bitshifter/glam-rs)
  * [https://github.com/bitshifter/mathbench-rs](https://github.com/bitshifter/mathbench-rs)

#### 性能測試

* [https://github.com/bitshifter/mathbench-rs](https://github.com/bitshifter/mathbench-rs)

## 系統程式

* [uutils / coreutils](https://github.com/uutils/coreutils): 以rust實作基本指令。
* [cargo-edit](https://crates.io/crates/cargo-edit): 可使用`cargo add $crate_name` 讓系統自動新增cargo.toml套件相依性。 `cargo rm $crate_name` 刪除cargo.toml的相依套件。
* [sudo-rs](https://github.com/memorysafety/sudo-rs)：以rust實作的sudo指令。

## 平行處理

* [Rayon: data parallelism in Rust](https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/)

## 非同步(異步)處理

* [async-std](https://github.com/async-rs/async-std)：最大的優點就是跟標准庫相容性較強。
* [tokio](https://tokio.rs/)：目前最受歡迎的異步運行時，功能強大，還提供了異步所需的各種工具(例如 tracing )、網絡協議框架(例如 HTTP，gRPC )等等。
  * [\[知乎\] 深入淺出Rust非同步程式設計之Tokio](https://zhuanlan.zhihu.com/p/107820568)
  * [\[知乎\] 深入瞭解 Rust 異步開發模式](https://zhuanlan.zhihu.com/p/104098627)
  * [\[知乎\] 揭開Rust Tokio的神秘面紗 | 第一篇 | 總覽](https://zhuanlan.zhihu.com/p/460984955)。
* [smol](https://github.com/smol-rs/smol)：smol 不同於 tokio 和 async-std，它更簡單和小巧。它致力於通過包裝和適配同步組件，將其異步化。有兩個主要特徵，採用線程池和 Async 裝飾器。
* [crossbeam](https://github.com/crossbeam-rs/crossbeam)：

[\[rust中文社區\] 麻煩問下 async-std future-rs tokio這三者的關系與不同](https://rustcc.cn/article?id=ab221876-5c51-47b7-9f16-647b2b8d290e)

簡單的說，std::future規定了標准，async-std、tokio等都是按標准做產品的廠家，這些廠家中tokio歷史最悠久，產品最可靠。

## 文字處理

* [serde - rust的多種序列化格式的解決方案](https://zhuanlan.zhihu.com/p/54004232)。
* [ripreg](https://github.com/BurntSushi/ripgrep)：文字搜尋工具，應該算是Rust的Cli殺手級應用。
* [itoa](https://crates.io/crates/itoa)：(非常快)快速的整數->字串套件。

## log處理

* [tokio-rs/tracing](https://github.com/tokio-rs/tracing)：強大的日誌框架，同時還支援OpenTelemetry格式，無縫打通未來的監控 rust-lang/log 官方日誌庫，事實上的API標准, 但是三方庫未必遵循。

## ORM

* [diesel-rs / diesel](https://github.com/diesel-rs/diesel)：高性能的ORM。
* [sfackler / rust-postgres](https://github.com/sfackler/rust-postgres)：純rust的postgres客戶端。

## GUI

* [tauri](https://github.com/tauri-apps/tauri)：
* [\[知乎\] Rust GUI 庫](https://zhuanlan.zhihu.com/p/278012049)

## TUI

* [indicatif](https://crates.io/crates/indicatif)：進度條。
* [ctrlc](https://crates.io/crates/ctrlc)：簡易 ctrl-c 處理程式。
* [clap](https://crates.io/crates/clap)：命令列參數解析器。
  * [clap-verbosity-flag](https://crates.io/crates/clap-verbosity-flag)：新增 --verbose 標籤到 structopt CLI。
* [structopt](https://crates.io/crates/structopt)：解析命令列參數為一個結構體。
* [atty](https://crates.io/crates/atty)：檢測應用程式是否運行在 tty 上。
* [exitcode](https://crates.io/crates/exitcode)：系統退出碼常數。

## 優化

* [bheisler / criterion.rs](https://github.com/bheisler/criterion.rs)：比官方提供的benchmark庫更好，目前已經成為事實上標准的效能測試工具。

## 繪圖

* [poloto](https://docs.rs/poloto/latest/poloto/)：以CSS控制SVG繪圖。
* [plotters](https://crates.io/crates/plotters/0.3.4)

## 金融

* [Panther](https://github.com/gregyjames/Panther): 以PyO3實作的技術分析函數，目前只有一些函數可用。
