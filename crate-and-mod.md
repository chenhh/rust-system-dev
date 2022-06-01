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

### 項目依賴

在Cargo.toml檔中，我們可以指定一個crate依賴哪些項目。這些依賴既可以是來自官方的crates.io，也可以是某個git倉庫位址，還可以是本地檔路徑。

```bash
[dependencies]
lazy_static = "1.0.0"
rand = { git = https://github.com/rust-lang-nursery/rand, branch = "master" }
my_own_project = { path = "/my/local/path", version = "0.1.0" }
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

### cargo.lock

當我們使用cargo build編譯完專案後，專案檔案夾內會產生一個新檔，名字叫Cargo.lock。它實際上是一個純文字檔，同樣也是toml格式。**它裡面記錄了當前專案所有依賴項目的具體版本。**

每次編譯專案的時候，如果該檔存在，cargo就會使用這個檔中記錄的版本號編譯專案；如果該檔不存在，cargo就會使用Cargo.toml檔中記錄的依賴專案資訊，自動選擇最合適的版本。

* 一般來說：如果我們的項目是庫，那麼最好不要把Cargo.lock檔納入到版本管理系統中，避免依賴庫的版本號被鎖死；
* 如果我們的項目是可執行程式，那麼最好要把Cargo.lock檔納入到版本管理系統中，這樣可以保證，在不同機器上編譯使用的是同樣的版本，生成的是同樣的可執行程式。



## prelude模組

除此之外，標準庫中的某些type、trait、function、macro等實在是太常用了。每次都寫use語句確實非常無聊，因此標準庫提供了一個std：：prelude模組，在這個模組中匯出了一些最常見的類型、trait等東西，編譯器會為用戶寫的每個crate自動插入一句話：

```rust
use std::prelude::*;
```

這樣，標準庫裡面的這些最重要的類型、trait等名字就可以直接使用，而無須每次都寫全稱或者use語句。Prelude模組的程式碼在[rust/library/std/src/prelude/mod.rs](https://github.com/rust-lang/rust/blob/master/library/std/src/prelude/mod.rs)資料夾下。目前的mod.rs中，直接匯出了v1模組中的內容，而v1.rs中，則是編譯器為我們自動導入的相關trait和類型。

