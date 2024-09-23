# Result

## 簡介

另一種錯誤處理方法，它也是一個枚舉類型，叫做Result，定義如下：

```rust
pub enum Result<T, E> {
    Ok(T),    // Ok(value) 表示操作成功，並包裝操作返回的 value
    Err(E),   // 表示操作失敗，並包裝 why，它（但願）能夠解釋失敗的原因
}
type Option<T> = Result<T, ()>;
```

在變體`Err(E)`為`None`時，`Option`可以被看作`Result`的特例，即<mark style="color:red;">`Option<T>=Result<T, ()>`</mark>。，`Err(E)`描述的是可能的錯誤而不是可能的不存在。

`Result`的處理方法和`Option`類似，都可以使用`unwrap`和`expect方`法，也可以使用`map`和`and_then`方法，並且用法也都類似。

如果你呼叫了 `Result::unwrap` 或 `Option::unwrap` ，`panic!`會分別在值為 Err 或 None 時發生，這用在程式碰到了無法回覆的錯誤。

在不需要處理錯誤的場景，例如寫原型、示例時，我們不想使用 match 去匹配 Result\<T, E> 以獲取其中的 T 值，因為 match 的窮盡匹配特性，你總要去處理下 Err 分支。簡化這個過程的方式就是 unwrap 和 expect。

## is\_ok, is\_err方法判斷變數是否成功

```rust
pub const fn is_ok(&self) -> bool {
        // 以matches巨集判斷*self之值是否為Ok
        matches!(*self, Ok(_))
    }
pub const fn is_err(&self) -> bool {
        !self.is_ok()
    }

// `is_ok` 和 `is_err` 判斷變數的結果為ok還是err
assert!(good_result.is_ok() && !good_result.is_err());
assert!(bad_result.is_err() && !bad_result.is_ok());
```

## 提取包含的值

當結果是 `Ok` 變體時，以下的這些方法可以提取 `Result<T, E>` 中包含的值。而如果 `Result` 是 `Err時`：

* `expect(msg)`： panics且帶有提供的自定義訊息。
* `unwrap()`： panics 帶有泛型資訊。
* `unwrap_or()`： 返回使用者提供的預設值。
* `unwrap_or_default`： 返回類型 `T` 的預設值 (必須實現 Default trait)。
* `unwrap_or_else` ：返回對提供的函數求值的結果。

### expect(msg)方法

```rust
// 返回包含 self 值的包含的 Ok 值。
// 如果值為 Err，就會出現 panics，其中 panic 訊息包括傳遞的訊息以及 Err 的內容。
pub fn expect(self, msg: &str) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed(msg, &e),
        }
    }

let x: Result<u32, &str> = Err("emergency failure");
x.expect("Testing expect"); // `Testing expect: emergency failure` 的 panics
```

### unwrap()方法

```rust
// 返回包含 self 值的包含的 Ok 值。
// 由於此函數可能為 panic，因此通常不建議使用該函數。只用於原型快速開發
// 如果該值為 Err，就會出現 Panics，並由 Err 的值提供 panic 訊息。
pub fn unwrap(self) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed("called `Result::unwrap()` on an `Err` value", &e),
        }
    }

let x: Result<u32, &str> = Ok(2);
assert_eq!(x.unwrap(), 2);
let x: Result<u32, &str> = Err("emergency failure");
x.unwrap(); // `emergency failure` 的 panics
```

### unwrap\_or(default)方法

```rust
// 返回包含的 Ok 值或提供的預設值。
pub fn unwrap_or(self, default: T) -> T {
        match self {
            Ok(t) => t,
            Err(_) => default,
        }
    }

let default = 2;
let x: Result<u32, &str> = Ok(9);
assert_eq!(x.unwrap_or(default), 9);

let x: Result<u32, &str> = Err("error");
assert_eq!(x.unwrap_or(default), default);
```

### unwrap\_or\_default()方法

```rust
// 返回包含的 Ok 值或默認值
//如果 Ok，則返回包含的值，否則如果 Err，則返回該類型的預設值。
 pub fn unwrap_or_default(self) -> T {
        match self {
            Ok(x) => x,
            Err(_) => Default::default(),
        }
    }
```

### unwrap\_or\_else(F)方法

```rust
// 返回包含的 Ok 值或從閉包中計算得出。
pub fn unwrap_or_else<F: FnOnce(E) -> T>(self, op: F) -> T {
        match self {
            Ok(t) => t,
            Err(e) => op(e),
        }
    }
```

## map, and then, or else

* `map` 通過將提供的函數應用於 `Ok` 的包含值並保持 `Err` 值不變，將 `Result<T, E>` 轉換為 `Result<U, E>`。

```rust
 pub fn map<U, F: FnOnce(T) -> U>(self, op: F) -> Result<U, E> {
        match self {
            Ok(t) => Ok(op(t)),
            Err(e) => Err(e),
        }
```

* `map_err` 通過將提供的函數應用於 `Err` 的包含值並保持 `Ok` 值不變，將 `Result<T, E>` 轉換為 `Result<T, F>`。

```rust
pub fn map_err<F, O: FnOnce(E) -> F>(self, op: O) -> Result<T, F> {
        match self {
            Ok(t) => Ok(t),
            Err(e) => Err(op(e)),
        }
    }
```

```rust
// `map` 消耗 `Result` 並產生另一個Result
let good_result: Result<i32, i32> = good_result.map(|i| i + 1);
let bad_result: Result<i32, i32> = bad_result.map(|i| i - 1);

// 使用 `and_then` 繼續計算
let good_result: Result<bool, i32> = good_result.and_then(|i| Ok(i == 11));

// 使用 `or_else` 處理該錯誤。
let bad_result: Result<i32, i32> = bad_result.or_else(|i| Ok(i + 20));
```

## 用match處理各類錯誤

例如Rust在`std::io`模塊定義了統一的錯誤類型Error，因此我們在處理時可以分別匹配不同的錯誤類型。

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        // 第一層處理Result的Ok與Err
        Ok(file) => file,
        Err(error) => match error.kind() {
            // 第二層針對不同的error列舉處理
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            ErrorKind::PermissionDenied => panic!("Permission Denied!"),
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

在處理`Result`時，我們還有一種處理方法，就是`try!`巨集(現在已被`?`運算子取代)。它會使代碼變得非常精簡，但是在發生錯誤時，會將錯誤返回，傳播到外部調用函數中，所以我們在使用之前要考慮清楚是否需要傳播錯誤。

```rust
use std::fs::File;

fn main() {
    let f = try!(File::open("hello.txt"));
}
```

`try!`使用起來雖然簡單，但也有一定的問題。像我們剛才提到的傳播錯誤，再就是有可能出現多層巢狀的情況。因此Rust引入了另一個語法糖來代替`try!`。它就是問號操作符“`?`”。

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn main() {
    read_username_from_file();
}

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    // 問號操作符必須在處理錯誤的程式碼後面
    f.read_to_string(&mut s)?;
    Ok(s)}
}
```

## unwrap - 故障時直接執行panic！

```rust
let p:Person = serde_json::from_str(data).unwrap();

// 近似行為
let p: Person = match serde_json::from_str(data) {
        Ok(p) => p,
        Err(e) => panic!("cannot parse JSON {:?}, e"), //panic
    }
```

如果我們可以確定輸入的json\_string始終會是可解析的，那麼使用unwrap沒有問題。但是如果會出現Err，那麼程式就會崩潰，無法從故障中恢復。在開發過程中，當我們更關心程式的主流程時，unwrap也可以作為快速原型使用。

因此unwrap隱含了panic!。雖然與更顯式的版本沒有差異，但是危險在於其隱含特性，因為有時這並不是你真正期望的行為。



## try! 巨集

在 `?` 運算子出現以前，相同的功能是使用 `try!` 巨集完成的。現在我們推薦使用 `?` 運算子，但是在老程式中仍然會看到 `try!`。

## ? - 故障時返回Err物件

Rust 中的問號 ( ?) 運算子用作返回`Result<T,E>`或`Option<T>`型別函式的錯誤傳播替代方案。?運算子是一種快捷方式，因為它減少了立即返回或從型別或函式中?返回所需的程式碼量。

當發生`Err`時，可以採取兩種行動：

1. `panic!`，不過我們已經決定要盡可能避免 panic 了。
2. 返回它，因為 Err 就意味著它已經不能被處理了。

? 幾乎就等於一個會返回 Err 而不是 panic 的 unwrap。<mark style="color:orange;">如果有個函式在它呼叫其它函式時發生了錯誤的情況，?算子它就把錯誤往上回傳</mark>。

<mark style="color:blue;">註：因為Result的回傳值為OK或Err，而Option的回傳值為Some或None，均為二類回傳值，可將?算子視為C/C++中的三元運算子，成功時傳回OK/Some，失敗時傳回Err/None</mark>。

錯誤傳播是傳播或返回程式碼中檢測到的錯誤資訊的過程，通常由呼叫函式觸發，以允許呼叫函式正確處理問題。當碰到Err時，我們不一定要panic!，也可以返回Err。不是每個Err都是不可恢復的，因此有時並不需要panic!。

```rust
let p: Person = match serde_json::from_str(data) {
        Ok(p) => p,
        Err(e) => return Err(e.into()),        // 失敗的話直接回傳錯誤
};

// 等價寫法，用p接Ok()的結果，或是Err()時直接回傳
// 可在使用p時，再針對Ok或是Err()的種類以match處理
let p:Person = serde_json::from_str(data)?;
```

使用match處理OK與Err很棒，因為：

* 在編寫程式碼時，您不會不小心忘記處理錯誤。
* 閱讀程式碼時，您可以立即看到這裡可能存在錯誤。

然而，它並不理想，因為它非常冗長。這就是問號運運算元的?用武之地。

加了?後，在函數回傳時，若OK，則會解壓Result至p中，若為Err時，會呼叫`Into::into`錯誤值以潛在地將其轉換為另一種型別。

以讀檔為例：

```rust
use std::io::{self, Read};

fn read_and_append<R: Read>(reader: R) -> io::Result<String> {
  let mut buf = String::new();
  match reader.read_to_string(&mut buf) {
    // 成功的話什麼都不用做
    Ok(_) => {}
    // 失敗的話直接回傳錯誤
    err => return err,
  }
  // 假設這邊還要做些處理後才回傳
  buf.push_str("END");
  // 回傳成功的結果
  Ok(buf)
}
```

其中的判斷錯誤，如果是錯誤就回傳的這段因為太常用到了，所以 Rust 就提供了個簡寫的方法，我們可以直接把上面的 match 那段改寫成：

```rust
reader.read_to_string(&mut buf)?;
```

如果它在成功時是會有回傳值的，比如 File::open 成功會回傳 File ，一個代表檔案的 struct ，那你也可以使用 ? 接住成功的結果。

```rust
let f = File::open("filename")?;
```

```rust
// 不使用?運算子的寫法
fn write_info(info: &Info) -> io::Result<()> {
    // 盡早返回錯誤
    let mut file = match File::create("my_best_friends.txt") {
           Err(e) => return Err(e),
           Ok(f) => f,
    };
    if let Err(e) = file.write_all(format!("name: {}\n", info.name).as_bytes()) {
        return Err(e)
    }
    if let Err(e) = file.write_all(format!("age: {}\n", info.age).as_bytes()) {
        return Err(e)
    }
    if let Err(e) = file.write_all(format!("rating: {}\n", info.rating).as_bytes()) {
        return Err(e)
    }
    Ok(())
}

// 使用?運算子的寫法
fn write_info(info: &Info) -> io::Result<()> {
    let mut file = File::create("my_best_friends.txt")?;
    // 盡早返回錯誤
    file.write_all(format!("name: {}\n", info.name).as_bytes())?;
    file.write_all(format!("age: {}\n", info.age).as_bytes())?;
    file.write_all(format!("rating: {}\n", info.rating).as_bytes())?;
    Ok(())
}
```

## 向上傳播錯誤

該函數的呼叫者最終會對 Result\<String,io::Error> 進行再處理，至於怎麼處理就是呼叫者的事，如果是錯誤，它可以選擇繼續向上傳播錯誤。

```rust
use std::fs::File;
use std::io::{self, Read};
/*
該函數返回一個 Result<String, io::Error> 類型，
當讀取使用者名稱成功時，返回 Ok(String)，
失敗時，返回 Err(io:Error)

File::open 和 f.read_to_string 返回的 
Result<T, E> 中的 E 就是 io::Error
*/
fn read_username_from_file() -> Result<String, io::Error> {
    // 打開檔案，f是`Result<檔案控制代碼,io::Error>`
    let f = File::open("hello.txt");

    let mut f = match f {
        // 打開檔案成功，將file控制代碼賦值給f
        Ok(file) => file,
        // 打開檔案失敗，將錯誤返回(向上傳播)
        Err(e) => return Err(e),
    };
    // 建立動態字串s
    let mut s = String::new();
    // 從f檔案控制代碼讀取資料並寫入s中
    match f.read_to_string(&mut s) {
        // 讀取成功，返回Ok封裝的字串
        Ok(_) => Ok(s),
        // 將錯誤向上傳播
        Err(e) => Err(e),
    }
}
```

可用?簡化。它的作用跟上面的 `match` 幾乎一模一樣。

如果結果是 `Ok(T)`，則把 `T` 賦值給 `f`，如果結果是 `Err(E)`，則返回該錯誤，所以 `?` 特別適合用來傳播錯誤。

雖然 ? 和 match 功能一致，但是事實上 ? 會更勝一籌.&#x20;

```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}

// ?運算子等效於以下程式
let mut f = match f {
    // 打開檔案成功，將file控制代碼賦值給f
    Ok(file) => file,
    // 打開檔案失敗，將錯誤返回(向上傳播)
    Err(e) => return Err(e),
};
```

想像一下，一個設計良好的系統中，肯定有自訂的錯誤特徵，錯誤之間很可能會存在上下級關係，例如標準庫中的 std::io::Error 和 std::error::Error，前者是 IO 相關的錯誤結構體，後者是一個最最通用的標準錯誤特徵，同時前者實現了後者，因此 std::io::Error 可以轉換為 std:error::Error。

明白了以上的錯誤轉換，? 的更勝一籌就很好理解了，它可以自動進行類型提升（轉換）：

```rust
// File::open 報錯時返回的錯誤是 std::io::Error 類型，
// 但是 open_file 函數返回的錯誤類型是 std::error::Error 的特徵物件，
// 可以看到一個錯誤類型通過 ? 返回後，變成了另一個錯誤類型
fn open_file() -> Result<File, Box<dyn std::error::Error>> {
    let mut f = File::open("hello.txt")?;
    Ok(f)
}
```

根本原因是在於標準庫中定義的 From 特徵，該特徵有一個方法 from，用於把一個類型轉成另外一個類型，? 可以自動呼叫該方法，然後進行隱式類型轉換。因此只要函數返回的錯誤 ReturnError 實現了 From 特徵，那麼 ? 就會自動把 OtherError 轉換為 ReturnError。

## 主函數main可傳回Result

我們所使用的所有 `main` 函數都返回 `()`。`main` 函數是特殊的因為它是可執行程式的入口點和退出點，為了使程式能正常工作，其可以返回的類型是有限制的，另一種就是傳回 `Result<(), E>`。

```rust
use std::error::Error;
use std::fs::File;

//Box<dyn Error> 類型是一個 trait物件，理解為 “任何類型的錯誤”
// 如果 main 返回 Ok(()) 可執行程式會以 0 值退出，
// 而如果 main 返回 Err 值則會以非零值退出；
fn main() -> Result<(), Box<dyn Error>> {
    let greeting_file = File::open("hello.txt")?;

    Ok(())
}
```

## 錯誤資訊類型不一樣，如何轉換？

```rust
fn foo() -> Result<i32, String> {
    // Err回傳類型為String
    Err(String::from("foo error"))
}

#[derive(Debug)]
struct MyErr{
    error_code: i32,
}

impl  From<String> for MyErr{
    fn from(s: String) -> Self {
        if s.is_empty(){
            MyErr{ error_code: 0}
        }else {
            MyErr{ error_code: 1}
        }
    }
}

fn bar() -> Result<i32, MyErr>{
    // Err回傳類型為i32
    // 錯誤資訊說，如果為 i32 實現了 From<String> 特性，還是允許的。
    foo()?;
    Ok(689)
}

fn main(){
    println!("{:?}", bar()); // Err(MyErr { error_code: 1 })
}
```
