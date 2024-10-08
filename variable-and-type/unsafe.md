# unsafe區塊

## 簡介

安全與非安全程式碼是靠 `unsafe` 關鍵字分離的，它扮演著兩種語言之間介面的角色。這也是我們理直氣壯地聲稱安全 Rust 是安全的原因：所有的非安全程式碼都被 `unsafe` 隔離在外。只要你願意，你甚至可以在程式碼根部新增`#![forbid(unsafe_code)]` 以保證你只會寫安全的程式碼。

有一些情況，編譯器的靜態檢查是不夠用的，它沒辦法自動推理出來這段程式碼究竟是不是安全的。這種時候，我們就需要使用unsafe關鍵字來保證程式碼的安全性。<mark style="color:red;">無論如何，安全 Rust 程式碼都不能導致未定義行為</mark>。

`unsafe` 關鍵字有兩層含義：<mark style="color:red;">聲明程式碼中存在編譯器無法檢查的安全規範，同時聲明開發者會自覺遵守相關規範而不會主動破壞它</mark>。

Rust的unsafe關鍵字有以下幾種用法：

* 用於修飾不安全函數fn；
* 用於修飾程式碼塊；
* 用於修飾不安全的trait；
* 用於修飾impl。
* 解引用裸指標。

unsafe區塊不意味著區塊中的程式碼就一定是危險的或者必然導致記憶體安全問題：其意圖在於作為開發者你將會確保 unsafe 塊中的程式碼以有效的方式訪問記憶體。

當一個fn是unsafe的時候，意味著我們在調用這個函數的時候需要非常小心。它可能要求調用者滿足一些其他的重要約束，而這些約束條件無法由編譯器自動檢查來保證。有unsafe修飾的函數，要麼使用unsafe語句塊調用，要麼在unsafe函數中調用。

**因此需要注意，unsafe函數是具有“傳遞性”的，unsafe函數的“調用者”也必須用unsafe修飾**。

```rust
// 函數簽名
pub unsafe fn from_raw_parts(buf: *mut u8, length: usize, capacity: usize) ->
String
// 自己保證這個緩衝區包含的是合法的 utf-8 字串
let s = unsafe { String::from_raw_parts(ptr as *mut _, len, capacity) } ;
```

使用`unsafe`關鍵字包圍起來的語句塊，裡面可以做一些一般情況下做不了的事情。但是，它也是有規矩的。與普通程式碼比起來，它多了以下幾項能力：

* 對裸指標執行解引用操作；
* 讀寫可變靜態變數；
* 讀union或者寫union的非Copy成員；
* 調用unsafe函數

在Rust中，有些地方必須使用unsafe才能實現。比如標準庫提供的一系列intrinsic函數，很多都是unsafe的，再比如調用extern函數必須在unsafe中實現。另外，一些重要的資料結構內部也使用了unsafe來實現一些功能。

\*\*當unsafe修飾一個trait的時候，那麼意味著實現這個trait也需要使用unsafe。\*\*比如在後面講執行緒安全的時候會著重講解的Send、Sync這兩個trait。因為它們很重要，是實現執行緒安全的根基，如果由程式師來告訴編譯器，強制指定一個類型是否滿足Send、Sync，那麼程式師自己必須很謹慎，必須很清楚地理解這兩個trait代表的含義，編譯器是沒有能力推理驗證這個impl是否正確的。這種impl對程式的安全性影響很大。

## 原始指標

Rust提供了兩種原始指標供我們使用，`*const T`和`*mut T`。我們可以通過`*mut T`修改所指向的資料，而`*const T`不能。在unsafe程式碼塊中它們倆可以互相轉換。

<mark style="color:red;">原始指標和C/C++中的指標功能相同，因此C/C++的程式碼可以用原始指標直接改寫(但是可能會有記憶體漏洞)</mark>。

### 原始指標與引用的比較

原始指標相對於其他的指針，如Box，&，\&mut來說，有以下區別：

* 裸指標可以為空，而且編譯器不保證裸指標一定指向一個合法的記憶體位址；
* 不會執行任何自動化清理工作，比如自動釋放記憶體等；
* 裸指標賦值操作執行的是簡單的記憶體淺複製，並且不存在借用檢查的限制。
* <mark style="color:red;">創建裸指標是完全安全的行為，只有對裸指標執行“解引用”才是不安全的行為</mark>，必須在unsafe語句塊中完成。

### 解引用一個原始指標

```rust
fn main() {
    let x = 1_i32;
    let mut y: u32 = 1;
    // 這是安全的
    let raw_mut = &mut y as *mut u32 as *mut i32 as *mut i64;
    unsafe {
        // 這是不安全的,必須在 unsafe 塊中才能通過編譯
        *raw_mut = -1;
    }
    println!("{:X} {:X}", x, y);
    // FFFFFFFF FFFFFFFF
}
```

<mark style="color:red;">我們可以把原始指標通過</mark><mark style="color:red;">`as`</mark><mark style="color:red;">運算子執行類型轉換。轉換類型之後，它就可以把它所指向的資料當成另外一個類型來操作了</mark>。原本變數y的類型是u32，但是我們對它取位址後，最後將指標類型轉換成了i64。此時，我們對該指標所指向的位址進行修改會發生“類型安全”問題以及“記憶體安全”問題。

可見，x原本是一個在堆疊上存在的不可變綁定，在我們通過<mark style="color:red;">原始</mark>指標對y做了修改之後，x的值也發生了變化。原因就是，我們對指向y的指標類型做了轉換，讓它以為自己指向的是i64類型，恰巧x就在y旁邊，城門失火，殃及池魚，x就被順帶一起修改了。從這個示例我們可以看到，unsafe程式碼中可以做很多危險的事情。

Rust的各種指標還有一些重要約束，比如`&mu`t型指標最多只能同時存在一個。這些約束條件，在unsafe場景下是很容易被打破的，而編譯器並沒有能力幫我們自動檢查出來。我們之所以需要unsafe，只是因為某些程式碼只有在特定條件下才是安全的，而這個條件我們沒有辦法利用類型系統表達出來，所以這時候需要依靠我們自己來保證。

### 讀寫一個可變的靜態變數static mut

```rust
static mut N: i32 = 5;
fn main() {
    unsafe {
        N += 1;
        println!("N: {}", N);  //6
    }
}
```

### 呼叫一個不安全函數

```rust
unsafe fn foo() {
    println!("unsafe func");
}
fn main() {
    unsafe {
        foo();
    }
}
```
