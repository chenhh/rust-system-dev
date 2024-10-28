# 錯誤處理

## 簡介

Rust 錯誤處理的手段主要分為兩種，對於不可恢復的錯誤（unrecoverable error），可以通過 panic 來直接中斷程式的執行，而對於可恢復的錯誤（recoverable error），一般會返回 Result 。

在實際開發中，一般程式中的<mark style="color:blue;">錯誤可以分為以下三類</mark>：

* <mark style="color:red;">**失敗**</mark>：違反規則的情況。比如函數參數需要的是字串，你傳入了數字類型，這是一種違反規則
* <mark style="color:red;">**錯誤**</mark>：一般是在意料中可能出現問題的地方出現了問題。比如http鏈接超時，訪問了不存在的檔案等。這些錯誤其實是開發者可以處理的。但是在那些異常處理語言中，把失敗和錯誤，都歸到了異常中，粒度非常粗。
* <mark style="color:red;">**異常**</mark>：真正的異常，是指完全不可預料的問題。比如NullError，訪問了越界的陣列等等，基本是開發者無法處理的情況。

<mark style="background-color:red;">Rust處理異常的方法有4種：Option、Result、Panic、Abort</mark>。

* 處理失敗：`Option<T>`，傳回值`Some(T)`，或者傳回`None`表示失數。
* 處理錯誤： `Result<T, E>`，傳回成功值`OK(T)`，或是錯誤`Err(E)`。
* 恐慌：`Panic!`巨集，發生恐慌執行緒即崩潰。比如記憶體用盡這種情況。也提供了catch\_unwind的方法來捕獲恐慌，恢復線程，但是為了內存安全，只有實現了UnwindSafe的類型才能安全catch。

在Safe Rust中基本可以保證異常安全，但是unsafe rust中則需要開發者來保證。即便如此，unsafe rust中也有種種限制不會隨便暴露記憶體。

Rust 並沒有提供基於 exception 的錯誤處理機制，雖然 panic! 巨集在讓行程掛掉時也拋出堆疊，同時也可以用 `std::panic::catch_unwind` 捕捉 panic，但是極其不推薦用來處理常規錯誤。

Rust 提供以下基礎設施做錯誤處理：

* Option, Result
* unwrap, expect
* combinators
* try! macro
* Error trait
* From trait
* Carrier trait
* try塊包住可能會出現的異常；
* 用catch將之捕獲；
* finally塊統一處理資源的清理；
* 對於自定義的函數，我們可以throw異常。

## Rust的Panic！

<mark style="color:red;">Rust裡沒有異常</mark>。 但如果非要和異常機制進行對映，Rust可以說做的相當徹底。

Rust把錯誤分成了兩大類：

* <mark style="color:purple;">一類是不可修復錯誤，建議使用panic來處理</mark>。對於不可修復錯誤，本質上沒有辦法在程式執行階段做好處理的，那麼就應該用panic讓程式主動退出，由開發者來修復程式碼，這是唯一合理的方案。
* <mark style="color:purple;">另外一類錯誤是可修復錯誤，一般使用返回值</mark><mark style="color:purple;">`Option<T>`</mark><mark style="color:purple;">或是</mark><mark style="color:purple;">`Result<T,E>`</mark><mark style="color:purple;">來處理</mark>。比如打開檔案出錯這種問題，應該是設計階段能預計到的，可以在執行階段更好處理的問題，就適合採用這種方案。

### 正常，以返回值的形式

相當於壓縮了上一節中的0、1、2項。沒有什麼情理中的意外，網路連不上、檔案找不到、非法輸入，統統都用返回值的方式。

### 致命錯誤，不可恢復，非崩潰不可。

* 一旦存在不可恢復的錯誤，Rust使用`Panic!`巨集來終止程式（執行緒）。一旦`Panic!`巨集出手，基本沒得救（`panic::catch_unwind`是個例外，稍後說）。執行時預設會進行stack unwind（堆疊反解），一層層上去，直到執行緒的頂端。
* 有些情況`panic!`是你的程式所依賴的函式庫產生的，比如陣列越界存取時的實現。
* 另一種情況，是你自己的程式邏輯判斷產生了不可恢復的錯誤，可以手動觸發`panic!`巨集來終止程式。此時`panic!`的使用與`throw`很類似。

## Rust的返回值Result

<mark style="color:red;">對於可恢復的錯誤，Rust一律使用返回值來進行檢查，而且提倡採用內建列舉</mark><mark style="color:red;">`Result`</mark>，還在實作層面給了一定的約束：<mark style="color:blue;">對於返回值為Result型別的函式，呼叫方如果沒有進行接收，編譯期會產生警告</mark>。很多庫函式都通過Result來告知呼叫方執行結果，讓呼叫方來決定是否嚴重到了使用Panic！的程度。

```cpp
enum Result<T, E>{
    Ok(T),    // T 代表成功時返回的 Ok 成員中的資料的類型
    Err(E),    // E 代表失敗時返回的 Err 成員中的錯誤的類型
}
```

在Rust標準庫中，可以找到許多以Result命名的型別，它們通常是Result泛型的特定版本，比如<mark style="color:blue;">File::open</mark>的返回值就是把T替換成了<mark style="color:blue;">std::fs::File</mark>，把E替換成了`std::io::Error`。

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("Problem opening the file: {:?}", error)
        },
    };
}
```

注意與 Option 枚舉一樣，Result 枚舉和其成員也被導入到了 prelude 中，所以就不需要在 match 分支中的 Ok 和 Err 之前指定 Result::。

當結果是 Ok 時，返回 Ok 成員中的 file 值，然後將這個檔案控制代碼賦值給變量 f。match 之後，我們可以利用這個檔案控制代碼來進行讀寫。

從 File::open 得到 Err 值的情況。在這種情況下，我們選擇調用 panic! 巨集。

### Error 類型

定義 Error 類型是一個可簡單，可複雜的事情，畢竟在 Result\<T, E> 裡，E 其實可以塞任何東西。甚至可以直接把 String 作為 Error 來使用(不建議)，還能帶上一定的錯誤資訊。

更多的時候，我們可能會想要把錯誤定義為一個 Enum 或者 Struct ，並實現 Error 等相關的 trait 。這是個體力活，如果你還需要處理 std 或者第三方庫拋出來的 Error ，還需要手工實現一大堆 From 來為自己的 Error 實現相應的轉換規則。

### 匹配不同的錯誤

如果 File::open 因為檔案不存在而失敗，我們希望創建這個檔案並返回新檔案的控制代碼。如果 File::open 因為任何其他原因失敗，例如沒有打開檔案的權限，我們仍然希望 panic!

[https://rustwiki.org/zh-CN/std/io/enum.ErrorKind.html](https://rustwiki.org/zh-CN/std/io/enum.ErrorKind.html)

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

io::ErrorKind 是一個標准庫提供的枚舉，它的成員對應 io 操作可能導致的不同錯誤類型。我們感興趣的成員是 ErrorKind::NotFound，它代表嘗試打開的檔案並不存在。

這樣，match 就匹配完 f 了，不過對於 error.kind() 還有一個內層 match。

我們希望在內層 match 中檢查的條件是 error.kind() 的返回值是否為 ErrorKind的 NotFound 成員。如果是，則嘗試通過 File::create 創建檔案。然而因為 File::create 也可能會失敗，還需要增加一個內層 match 語句。當檔案不能被打開，會列印出一個不同的錯誤資訊。外層 match 的最後一個分支保持不變，這樣對任何除了檔案不存在的錯誤會使程式 panic。

### panic::catch\_unwind

```cpp
use std::panic;
​
let result = panic::catch_unwind(|| {
    println!("hello!");
});
assert!(result.is_ok());
​
let result = panic::catch_unwind(|| {
    panic!("oh no!");
});
assert!(result.is_err());
```

沒錯，它的行為幾乎就是try/catch了：panic！巨集被捕獲了，程式並也沒有掛，返回了Err。儘管如此，Rust的目的並不是讓它成為try/catch機制的實現，而是當Rust和其他程式語言互動時，避免其他語言程式碼塊throw出異常。所以呢，錯誤處理的正道還是用Result。

從catch\_unwind的名字上，需要留意下unwind這個限定詞，它意味著只有預設進行堆疊反解的panic可以被捕獲到，如果是設為直接終止程式的panic，就逮不住了。

catch\_unwind 一般是用來在多執行緒程式裡面在將掛掉的執行緒 catch 住，防止一個執行緒掛掉導致整個進程崩掉，或者是通過外部函數介面(FFI)與 C 互動時將堆疊資訊兜住防止 C 程式看到堆疊不知道如何處理，直接把堆疊資訊丟給 C 程式的話屬於 C 裡的未定義行為(Undefined Behavior)。

另外 catch\_unwind 並不保證能 catch 所有 panic，而只對通過 unwind 實現的 panic 有用。因為 unwind 需要額外記錄堆疊資訊，對程式效能和二進製程式大小有影響，所以在一些嵌入式平臺上面的 panic 並沒有通過 unwind 實現，而是直接 abort 的，所以 catch\_unwind 並不保證能捕捉到所有panic。

## 基本錯誤處理

Rust 錯誤處理本質上還是基於返回值的，很多基於返回值做錯誤處理的語言是將錯誤直接硬編碼到正確值上，或者返回兩個值，前者例如 C 在很多時候都是直接把正常情況永遠不會出現的值作為錯誤值，後者例如 Go 同時返回兩個值來進行錯誤處理。

而 Rust 則將兩個可能的值用 enum 類型表示，enum 在函數式語言裡面叫做代數資料類型(algebraic data type)，而且是和類型(sum type)，表示兩個可能的值一次只能取一個。

```cpp
enum Option<T> {
    None,
    Some(T),    // struct tuple
}

enum Result<T, E> {
    Ok(T),    // struct tuple
    Err(E),    // struct tuple
}
```

Rust用於錯誤處理的最基本的類型就是我們常見的Option\<T>類型。<mark style="color:red;">**用 Option\<T> 表示錯誤時一般不關心錯誤原因，出錯時直接返回空值 None**</mark> 。一般就直接用 Option\<T> 例如從 HashMap 裡面取值或者對 Vector 進行 pop 操作，前者出錯了只可能是對應的 key 不存在，後者出錯只可能是 Vector 已經是空的了。

<mark style="color:red;">而 Result\<T, E> 則將錯誤的不同原因包括進來了，</mark><mark style="color:red;background-color:red;">Option\<T> 相當於 Result\<T, ()></mark>。而如果錯誤可能是多種原因造成的則用 Result\<T, E> 來表示，例如 IO 錯誤，原因可能是 NotFound, PermissionDenied, AlreadyExists, InvalidData…。

在看各種文檔或讀別人的程式碼時發現 Result\<T> 的錯誤類型時可能會有點小疑惑。因為Result\<T> 是用到了類型別名。

例如 io::Result 定義如下，因為 io 模組裡的錯誤都是 io::Error，用到 Result\<T, Error> 的地方如果都換成 Result\<T> 會少敲很多次鍵盤，同時又不會產生歧義(因為 Result 裡面的 E 已經被固定成 io::Error 了)

```rust
type Result<T> = Result<T, Error>;
```

## 丟失上下文

僅僅只是把 Error 定義出來只不過是剛剛踏入了錯誤處理的門，甚至可以說定義 Error 也只是錯誤處理那一系列 boilerplate code 的一小部分而已。單純見到錯誤就往上拋並不難，而且 Rust 還提供了 ? 運算子來讓你可以更爽地拋出錯誤，但與之相對的，直接上拋錯誤，就意味著丟棄了大部分錯誤的上下文，也會給時候定位問題帶來不便。

```rust
/*
eat()/drink()/work()/sleep() 中任意一個都有可能拋出 io::Error 的函數。
那麼當 daily() 出錯時，你拿到的最終資訊可能只是個 
"I/O error: failed to fill whole buffer" ，而到底是哪裡出的錯，
為什麼出錯了呢？不知道，因為錯誤來源丟失了。
*/

fn daily() -> Result<(), MyError> {
    eat()?;
    drink()?;
    work()?;
    sleep()?;
    Ok(())
}
```

為了避免出現類似的問題，遇到錯誤時就需要注意儲存一些呼叫資訊以及錯誤的現場，概況下來，就是兩樣東西

* 呼叫堆疊，或者說 backtrace。
* 錯誤的上下文，如關鍵參數。

光靠 backtrace 其實只能回答哪裡出了錯的問題，而回答不了為什麼出錯的。一些預期內時常會拋錯誤的程式碼路徑也不宜獲取 backtrace。

[https://juejin.cn/post/6940446226786156575](https://juejin.cn/post/6940446226786156575)

## 參考資料

* [\[知乎\] Rust 錯誤處理](https://zhuanlan.zhihu.com/p/25506762)
