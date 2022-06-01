# Result

## 簡介

另一種錯誤處理方法，它也是一個枚舉類型，叫做Result，定義如下：

```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

在變體`Err(E)`為`None`時，`Option`可以被看作`Result`的特例。

`Result`的處理方法和`Option`類似，都可以使用`unwrap`和`expect方`法，也可以使用`map`和`and_then`方法，並且用法也都類似。

如果你呼叫了 `Result::unwrap` 或 `Option::unwrap` ，`panic!`會分別在值為 Err 或 None 時發生，這用在程式碰到了無法回覆的錯誤。

## 用match處理各類錯誤

例如Rust在`std::io`模塊定義了統一的錯誤類型Error，因此我們在處理時可以分別匹配不同的錯誤類型。

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        // 第一層處理Result的兩個變體
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

在處理Result時，我們還有一種處理方法，就是try!巨集。它會使代碼變得非常精簡，但是在發生錯誤時，會將錯誤返回，傳播到外部調用函數中，所以我們在使用之前要考慮清楚是否需要傳播錯誤。

```rust
use std::fs::File;

fn main() {
    let f = try!(File::open("hello.txt"));
}
```

try!使用起來雖然簡單，但也有一定的問題。像我們剛才提到的傳播錯誤，再就是有可能出現多層巢狀的情況。因此Rust引入了另一個語法糖來代替try!。它就是問號操作符“?”。

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

## ? - 故障時返回Err物件

Rust 中的問號 ( ?) 運算子用作返回`Result<T,E>`或`Option<T>`型別函式的錯誤傳播替代方案。?運算子是一種快捷方式，因為它減少了立即返回或從型別或函式中?返回所需的程式碼量。

<mark style="color:orange;">如果有個函式在它呼叫其它函式時發生了錯誤的情況，?算子它就把錯誤往上回傳</mark>。

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

## 返回值由Option 轉 Result

`ok_or()` 可以把 Option 類型轉換為 Result。

```rust
fn foo() -> Option<i32> {
    None
}

fn bar() -> Result<i32, String> {
    foo().ok_or("error".to_string())?;
    Ok(689)
}

fn main() {
    println!("{:?}", bar()); //Err("error")
}
```

## 返回值由Result轉Option

如果想提前丟棄計算結果，可以用 `.ok()?` 方法。

```rust
fn foo() -> Result<i32, i32> {
    Err(123)
}

fn bar() -> Option<i32> {
    foo().ok()?;
    Some(689)
}

fn main() {
    println!("{:?}", bar()); //None
}
```

