# crate與mod

## 簡介

Rust用了兩個概念來管理專案：一個是crate，一個是mod。

* crate簡單理解就是一個項目 \(project\)。crate是Rust中的獨立編譯單元。每個crate對應生成一個庫或者可執行檔（如.lib.dll.so.exe等）。
  * 官方有一個crate倉庫 https://crates.io/，可以供用戶發佈各種各樣的庫，用戶也可以直接使用這裡面的開源庫。
* mod簡單理解就是命名空間。mod可以嵌套，還可以控制內部元素的可見性。
* **crate和mod有一個重要區別是：crate之間不能出現迴圈引用；而mod是無所謂的，mod1要使用mod2的內容，同時mod2要使用mod1的內容，是完全沒問題的**。

在Rust裡面，crate才是一個完整的編譯單元（compile unit）。也就是說，rustc編譯器必須把整個crate的內容全部讀進去才能執行編譯，rustc不是基於單個的.rs檔或者mod來執行編譯的。作為對比，C/C++裡面的編譯單元是單獨的.c/.cpp檔以及它們所有的include檔。每個.c/.cpp檔都是單獨編譯，生成.o檔，再把這些.o檔連結起來。

Rust有一個極簡標準庫，叫作std，除了極少數嵌入式系統下無法使用標準庫之外，絕大部分情況下，我們都需要用到標準庫裡面的東西。為了效率，Rust編譯器對標準庫有特殊處理。**預設情況下，使用者不需要手動添加對標準庫的依賴，編譯器會自動引入對標準庫的依賴**。

## cargo

{% embed url="https://doc.rust-lang.org/cargo/" %}

```bash
USAGE:
    cargo [+toolchain] [OPTIONS] [SUBCOMMAND]

OPTIONS:
    -V, --version                  Print version info and exit
        --list                     List installed commands
        --explain <CODE>           Run `rustc --explain CODE`
    -v, --verbose                  Use verbose output (-vv very verbose/build.rs output)
    -q, --quiet                    No output printed to stdout
        --color <WHEN>             Coloring: auto, always, never
        --frozen                   Require Cargo.lock and cache are up to date
        --locked                   Require Cargo.lock is up to date
        --offline                  Run without accessing the network
        --config <KEY=VALUE>...    Override a configuration value (unstable)
    -Z <FLAG>...                   Unstable (nightly-only) flags to Cargo, see 'cargo -Z help' for details
    -h, --help                     Prints help information

Some common cargo commands are (see all commands with --list):
    build, b    Compile the current package
    check, c    Analyze the current package and report errors, but don't build object files
    clean       Remove the target directory
    doc, d      Build this package's and its dependencies' documentation
    new         Create a new cargo package
    init        Create a new cargo package in an existing directory
    run, r      Run a binary or example of the local package
    test, t     Run the tests
    bench       Run the benchmarks
    update      Update dependencies listed in Cargo.lock
    search      Search registry for crates
    publish     Package and upload this package to the registry
    install     Install a Rust binary. Default location is $HOME/.cargo/bin
    uninstall   Uninstall a Rust binary
```

Cargo是Rust的包管理工具，是隨著編譯器一起發佈的。在使用rustup安裝了官方發佈的Rust開發套裝之後，Cargo工具就已經安裝好了，無須單獨安裝。

cargo只是一個包管理工具，並不是編譯器。Rust的編譯器是rustc，使用cargo編譯工程實際上最後還是調用的rustc來完成的。如果我們想知道cargo在後面是如何調用rustc完成編譯的，可以使用cargo build--verbose選項查看詳細的編譯命令。

* `cargo new hello_world --bin`，建立名稱為hello\_world可執行檔專案。
* `cargo new hello_world --lib`，建立名稱為hello\_world函式庫專案。
* 程式碼在`{PROJ_DIR}/src`資料夾中
  * `carge build`編譯產生的檔案在`{PROJ_DIR}/target/debug`資料夾中。
  * `cargo build release` 編譯產生的檔案在 `{PROJ_DIR}/target/release` 資料夾中。
* `cargo run` 可以執行debug版的程式，`cargo run --release`可以執行release版的程式。
* cargo只是一個包管理工具，並不是編譯器。Rust的編譯器是rustc，使用cargo編譯工程實際上最後還是調用的rustc來完成的。如果我們想知道cargo在後面是如何調用rustc完成編譯的，可以使用`cargo build--verbose`選項查看詳細的編譯命令。
* `cargo check` ，可以只檢查編譯錯誤，而不做程式碼優化以及生成可執行程式，非常適合在開發過程中快速檢查語法、類型錯誤。
* `cargo clean`，清理以前的編譯結果。
* `cargo doc`，生成文件。
* `cargo test`，生成單元測試。
* `cargo bench`，執行性能測試。
* `cargo update`，升級所有依賴項的版本，重新生成Cargo.lock文件。
* `cargo install`，安裝rust可執行程式。它可以讓用戶自己擴展cargo的子命令，為它增加新功能。
  * 例：`cargo install cargo-tree`增加子命令後，可使用`cargo tree`印出依賴樹模型。
  * 在crates.io網站上，用subcommand關鍵字可以搜出許多有用的子命令，用戶可以按需安裝。
* `cargo uninstall`，刪除rust可執行程式。

### 項目依賴

在Cargo.toml檔中，我們可以指定一個crate依賴哪些項目。這些依賴既可以是來自官方的crates.io，也可以是某個git倉庫位址，還可以是本地檔路徑。

```bash
[dependencies]
lazy_static = "1.0.0"
rand = { git = https://github.com/rust-lang-nursery/rand, branch = "master" }
my_own_project = { path = "/my/local/path", version = "0.1.0" }
```

## 使用函數庫

### 自建函式庫

* `cargo new good_bye --lib`
* `{PROJ_DIR}/src/lib.rs`是函式庫的入口檔案，

```rust
// lib.rs
pub fn say() {
    println!("good bye");
}
```

* 希望hello\_world專案能引用good\_bye專案。打開hello\_world項目的Cargo.toml文件，在依賴項下面添加對good\_bye的引用

```bash
[dependencies]
good_bye = { path = "../good_bye" }
```

現在在應用程式中調用這個庫。打開main.rs原始檔案，修改程式碼後，使用`cargo run`編譯並執行。

```rust
extern crate good_bye;
fn main() {
    println!("Hello, world!");
    good_bye::say();
}
```



### crate版本號

Rust裡面的crate都是自帶版本號的。基本意思如下：

* 1.0.0以前的版本是不穩定版本，如果出現了不相容的改動，升級次版本號，比如從0.2.15升級到0.3.0；
* 在1.0.0版本之後，如果出現了不相容的改動，需要升級主版本號，比如從1.2.3升級到2.0.0；
* 在1.0.0版本之後，如果是相容性的增加API，雖然不會導致下游用戶編譯失敗，但是增加公開的API情況，應該升級次版本號。

### 在\[dependencies\]裡面的幾種依賴項的格式

#### 来自crates.io的依賴

絕大部分優質開源庫，作者都會發佈到官方倉庫中，所以我們大部分的依賴都是來自於這個地方。在crates.io中，每個庫都有一個獨一無二的名字，我們要依賴某個庫的時候，只需指定它的名字及版本號即可。

```rust
[dependencies]
lazy_static = "1.0"
```

指定版本號的時候，可以使用模糊匹配的方式。

* ^符號，如^1.2.3代表1.2.3&lt;=version&lt;2.0.0；
* ~符號，如~1.2.3代表1.2.3&lt;=version&lt;1.3.0；
* \*符號，如1.\*代表1.0.0&lt;=version&lt;2.0.0；
* 比較符號，比如&gt;=1.2.3、&gt;1.2.3、&lt;2.0.0、=1.2.3含義基本上一目了然。
* 還可以把多個限制條件合起來用逗號分開，比如version="&gt;1.2，  &lt;1.9"。

直接寫一個數位的話，等同於^符號的意思。所以lazy\_static="1.0"等同於lazy\_static="^1.0"，含義是1.0.0&lt;=version&lt;2.0.0。cargo會到網上找到當前符合這個約束條件的最新的版本下載下來。

#### 來自git倉庫的依賴

除了最簡單的git="…"指定repository之外，我們還可以指定對應的分支。或者指定當前的commit號。

```bash
# 指定分支
rand = { git = https://github.com/rust-lang-nursery/rand, branch = "next" }

# 指定commit號
rand = { git = https://github.com/rust-lang-nursery/rand, branch = "master", rev =
"31f2663" }

# 指定對應的tag名字
rand = { git = https://github.com/rust-lang-nursery/rand, tag = "0.3.15" }
```

#### 來自本地檔路徑的依賴

指定本地檔路徑，既可以使用絕對路徑也可以使用相對路徑。

### cargo.toml

Cargo.toml是我們的專案管理設定檔，這裡記錄了該專案相關的元資訊。

### cargo.lock

當我們使用cargo build編譯完專案後，專案檔案夾內會產生一個新檔，名字叫Cargo.lock。它實際上是一個純文字檔，同樣也是toml格式。**它裡面記錄了當前專案所有依賴項目的具體版本。**

每次編譯專案的時候，如果該檔存在，cargo就會使用這個檔中記錄的版本號編譯專案；如果該檔不存在，cargo就會使用Cargo.toml檔中記錄的依賴專案資訊，自動選擇最合適的版本。

* 一般來說：如果我們的項目是庫，那麼最好不要把Cargo.lock檔納入到版本管理系統中，避免依賴庫的版本號被鎖死；
* 如果我們的項目是可執行程式，那麼最好要把Cargo.lock檔納入到版本管理系統中，這樣可以保證，在不同機器上編譯使用的是同樣的版本，生成的是同樣的可執行程式。

## 預設配置

cargo也支持設定檔。設定檔可以定制cargo的許多行為，就像我們給git設置設定檔一樣。類似的，cargo的設定檔可以存在多份，它們之間有優先順序關係。

你可以為某個資料夾單獨提供一份設定檔，放置到當前資料夾的.cargo/config位置，也可以提供一個全域的預設配置，放在$HOME/.cargo/config位置。

## workspace

cargo的workspace概念，是為了解決多crate的互相協調問題而存在的。假設現在我們有一個比較大的項目。我們把它拆分成了多個crate來組織，就會面臨一個問題：不同的crate會有各自不同的Cargo.toml，編譯的時候它們會各自產生不同的Cargo.lock檔，我們無法保證所有的crate對同樣的依賴項使用的是同樣的版本號。

為了讓不同的crate之間能共用一些資訊，cargo提供了一個workspace的概念。一個workspace可以包含多個專案；所有的項目共用一個Cargo.lock檔，共用同一個輸出目錄；一個workspace內的所有項目的公共依賴項都是同樣的版本，輸出的目的檔案都在同一個資料夾內。workspace同樣是用Cargo.toml來管理的。我們可以把所有的項目都放到一個資料夾下面。在這個資料夾下寫一個Cargo.toml來管理這裡的所有專案。

Cargo.toml檔中要寫一個\[workspace\]的配置：

```rust
[workspace]
members = [
"project1", "lib1"
]
```

```rust
├── Cargo.lock
├── Cargo.toml
├── project1
│ ├── Cargo.toml
│ └── src
│ └── main.rs
├── lib1
│ ├── Cargo.toml
│ └── src
│ └── lib.rs
└── target
```

我們可以在workspace的根目錄執行cargo build等命令。請注意，雖然每個crate都有自己的Cargo.toml檔，可以各自配置自己的依賴項，但是每個crate下面不再會各自生成一個Cargo.lock檔，而是統一在workspace下生成一個Cargo.lock文件。如果多個crate都依賴一個外部庫，那麼它們必然都是依賴的同一個版本。

## prelude模組

除此之外，標準庫中的某些type、trait、function、macro等實在是太常用了。每次都寫use語句確實非常無聊，因此標準庫提供了一個std：：prelude模組，在這個模組中匯出了一些最常見的類型、trait等東西，編譯器會為用戶寫的每個crate自動插入一句話：

```rust
use std::prelude::*;
```

這樣，標準庫裡面的這些最重要的類型、trait等名字就可以直接使用，而無須每次都寫全稱或者use語句。Prelude模組的程式碼在[rust/library/std/src/prelude/mod.rs](https://github.com/rust-lang/rust/blob/master/library/std/src/prelude/mod.rs)資料夾下。目前的mod.rs中，直接匯出了v1模組中的內容，而v1.rs中，則是編譯器為我們自動導入的相關trait和類型。

