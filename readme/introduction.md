# Rust簡介

## 簡介

Rust語言是一門系統程式設計語言，它有三大特點：**執行快、防止段錯誤、保證執行緒安全**。

系統級程式設計是相對于應用級程式設計而言。一般來說，系統級程式設計意味著更底層的位置，它更接近於硬體層次，並為上層的應用軟體提供支援。系統級程式設計語言一般具有以下特點：

* 可以在資源非常受限的環境下執行；
* 執行時開銷很小，非常高效；
* ·很小的執行庫，甚至於沒有；
* 可以允許直接的記憶體操作。

目前，C和C++應該是業界最流行的系統程式設計語言。Rust的定位與它們類似，但是增加了安全性。C和C++都是編譯型語言，無須規模龐大的執行時（runtime）支援，也沒有自動記憶體回收（Garbage Collection）機制。

### 資料夾

不論是window或linux，安裝後的工具位於`${home}/.cargo/bin`資料夾中。

rustup的目錄在`$RUST_HOME`中，預設為：

* windows安裝後，指令位於`C:\Users\$USER\.rustup\toolchains\stable-x86_64-pc-windows-msvc\bin`的資料夾中。
* Linux安裝後，指令位於`$HOME/.rustup/toolchain/stable-x86_64-unknown-linux-gnu/bin`中。

1.61版本目前指令有：

* `cargo`: rust的軟體包管理器。
* `cargo-clippy`: 檢查軟體包裡的rust語法錯誤與改善品質的軟體。
* `cargo-fmt`: 使用rust-fmt格式化bin/lib中所有rust的語法與排版。
* `clippy-driver`: 同cago-clippy。
* evcxr：
* `rls`：
* `rustc`: rust編譯器。
* `rustdoc`
* `rustfmt`：格式化rust原始碼。
* `rust-gdb`: rust除錯器，命令列模式。
* `rust-gdbgui`：rust除錯器，圖形化模式。
* `rust-lldb`：
* `rustup`：
* `wasm-pack`：

### cargo命令

* 檢視版本：`cargo --version`
* 構建專案：`cargo build`
* 執行專案：`cargo run`
* 測試專案：`cargo test`
* 為專案構建檔案：`cargo doc`
* 將庫發布到 crates.io：`cargo publish`

cargo命令詳見[crate與cargo](../crate-and-cargo/)章節。

### rustup管理工具鏈(toolchain)

Rust 以六星期為週期進行快速的版本發行，且支援多個作業系統。有三個頻道(channels): nightly, beta, stable。

* 顯示目前已安裝與使用中的工具鏈頻道: `rustup show`
  * 單獨顯示目前使用的工具鏈頻道：`rustup show active-toolchain`
  * 顯示目前rustup的目錄：`rustup show home`
  * 顯示目前的profile： `rustup show profile`
  * 顯示目前的rust pgp公鑰： `rustup show keys`
* rust版本直接更新，可單獨指定頻道或特定版本：`rustup update`
* 檢查是否有可用的更新：`rustup check`
* 指定預設的工具鏈頻道或版本：`rustup default {stable|beta|nightly}`
* 顯示目前已下載的工具鏈： `rustup toolchain list`
  * 安裝工具鏈：`rustup toolchain install {stable|beta|nightly}`
  * 移除工具鏈：`rust toolchain uninstall {stable|beta|nightly}`
* 安裝頻道: `rustup install {stable|beta|nightly}`
* 切換工具鏈: `rustup override set {stable|beta|nightly}`
* 反安裝: `rustup self uninstall`

## 記憶體安全(memory safe)

Rust最重要的特點就是可以提供記憶體安全保證，而且沒有額外的性能損失。

在傳統的系統級程式設計語言（C/C++）的開發過程中，經常出現因各種記憶體錯誤引起的崩潰或bug。比如空指標、野指標、記憶體洩漏、記憶體越界、段錯誤、資料競爭、迭代運算器失效等。

鑒於手動記憶體管理非常容易出問題，發明了一種自動垃圾回收的機制（Garbage Collection），因此程式設計師在絕大多數情況下不用再操心記憶體釋放的問題。新發明的絕大多數程式設計語言都使用了基於各種高級演算法的自動垃圾回收機制，因為它確實方便，使大家能更專注於業務邏輯的部分。但是到目前為止，不管使用哪種演算法的GC系統，在性能上都要付出比較大的代價。要麼需要較大的執行時佔用較大記憶體，要麼需要暫停整個程式，要麼具備不確定性的時延。如果在性能敏感的領域，這是完全不可接受的。

這些年來，許多新的語言特性被發明出來，許多優秀的程式設計範式被總結出來，許多高品質的程式碼庫被開發出來。但是記憶體安全問題依然沒有很好的被解決，再多的努力，也只能減少它出現的機會，很難保證完整地解決掉這一類錯誤。

Rust就是為此而生的。Rust語言是可以保證記憶體安全的系統級程式設計語言。這是它的獨特的優勢。

## 併行(concurrency)

在傳統的系統級程式設計語言中，併行程式碼很容易出錯，而且有些問題很難複現，難以發現和解決問題，除錯的成本非常高。執行緒安全問題一直以來都是非常令人頭痛的問題。

在強大的記憶體安全特性的支援下，Rust一舉解決了併發條件下的資料競爭（Data Race）問題。它從編譯階段就將資料競爭解決在了萌芽狀態，保障了執行緒安全。Rust在併發方面還具有相當不錯的可擴展性。所有跟執行緒安全相關的特性，都不是在編譯器中寫死的。用戶可以用函式庫的形式實現各種高效且安全的併發程式設計模型，進而充分利用多核時代的硬體性能。

## 版本與發佈策略

版本格式為：主版本號.次版本號.修訂號。

版本號遞增規則如下：

* 主版本號：當你做了不相容的API修改。
* 次版本號：當你做了向下相容的功能性新增。
* 修訂號：當你做了向下相容的問題修正。

Rust的第一個正式版本號是1.0，是2015年5月發佈的。從那以後，只要版本沒有出現大規模的不相容的升級，大版本號就一直維持在“1”，而次版本號會逐步升級。Rust一般以6個星期更新一個正式版本的速度進行反覆升級。

Rust使用了多管道發佈的策略：**nightly、beta、stable**版本。

* nightly版本是每天在主版本上自動創建出來的版本，這個版本上的功能最多，更新最快，但是某些功能存在問題的可能性也更大。因為新功能會首先在這個版本上開啟，供用戶試用。
* beta版本是每隔一段時間，將一些在nightly版本中驗證過的功能開放給用戶使用。它可以被看作stable版本的“預發佈”版本。
* 而stable版本則是正式版，它每隔6個星期發佈一個新版本，一些實驗性質的新功能在此版本上無法使用。它也是最穩定、最可靠的版本。stable版本是保證向前相容的。

Rust語言相對重大的設計，必須經過RFC（Request For Comments）設計步驟。這個步驟主要是用於討論如何“設計”語言。這個專案存在於[https://github.com/rust-lang/rfcs](https://github.com/rust-lang/rfcs)。所有大功能必須先寫好設計文檔，講清楚設計的目標、實現方式、優缺點等，讓整個社區參與討論，然後由“核心組”（Core Team）的成員參與定奪是否接受這個設計。

Rust語言每個相對複雜一點的新功能，都要經歷如下步驟才算真正穩定可用：RFC->nightly->beta->stable。

先編寫一份RFC，其中包括這個功能的目的、詳細設計方案、優缺點探討等。如果這個RFC被接受了，下一步就是在編譯器中實現這個功能，在nightly版本中開啟。經過幾個星期甚至幾個月的試用之後，根據回饋結果來決定撤銷、修改或者接受這個功能。如果表現不錯，它就會進入beta版本，繼續過幾個星期後，如果確實沒發現什麼問題，最終會進入stable版本。至此，這個功能才會被官方正式定為“穩定的”功能，在後續版本中要確保相容性的。

在2017年下半年，Rust設計組又提出了一個基於epoch的演進策略（後來也被稱為edition）。它要解決的問題是，如何讓Rust更平穩地進化。比如，有時某些新功能確實需要一定程度上破壞相容性。。簡單來說就是讓Rust的相容性保證是一個有時限的長度，而不是永久。Rust設計組很可能會在不久的將來發佈一個2018edition，把之前的版本叫作2015 edition。在這個版本的進化過程中，就可以實施一些不相容的改變。

## Rust能夠解決以及可能出現的bug

以下是Rust程式可以出現的bugs：

* deadlock
* logic bug
* memory leak
* fail to call destructors （出現循環引用的時候）
* race condition
* overflow integer

以下是Rust程式在safe域裡面不可能出現的bug：

* data race
* undefined behavior (如use-after-free, dangling-pointer, double-free, access violation)

unsafe域裡面一切皆有可能（包括safe與unsafe的結合）。因為真正的程式總會有各種困難的問題，不可能通過語言層面解決。但是Rust通過區分safe和unsafe將問題進行隔離。

## 參考資料

[\[知乎\] 如何看待 Rust 這門語言？](https://www.zhihu.com/question/432640008/answer/1668000615)

[This Week in Rust](https://this-week-in-rust.org/)

[why-is-my-rust-build-so-slow](https://fasterthanli.me/articles/why-is-my-rust-build-so-slow)
