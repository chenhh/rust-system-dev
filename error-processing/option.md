# Option

## 簡介

Option是一種枚舉類型，主要包括兩種值：`Some(T)`和`None`，強制使用者判斷是否存在空值。

```rust
enum Option<T> {
    Some(T),    // Some表示有值
    None,       // None表示空值
}
```

[\[rust-doc\] std::option](https://doc.rust-lang.org/std/option/index.html) ([中文](https://rustwiki.org/zh-CN/std/option/index.html))。

因為Option為為枚舉，可用match處理分支狀態，且要處理所有的情況。

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    // 用match處理Option的分支
    // 注意=> 後面的表達式都沒有分號; 
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}
```

## 單層Option取值的方法

一般用兩種方法取值，一種是[`expect(custom error msg)`](https://doc.rust-lang.org/std/option/enum.Option.html#method.expect)，可自定義錯誤時的訊息。另一種是[`unwrap()`](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap)系列的方法(unwrap, [unwrap\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap\_or), [unwrap\_or\_default](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap\_or\_default)(default),[ unwrap\_or\_else](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap\_or\_else)(fn), [unwrap\_unchecked](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap\_unchecked)())，錯誤時直接panic!。

```rust
// 由unwrap()的定義可以看出是直接取值，或在None時直接panic
pub const fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic("called `Option::unwrap()` on a `None` value"),
        }
    }

// 由unwrap()的定義可以看出是直接取值，或在None時傳回給定的default值
// const trait中所有其中的method都必須是const?
pub const fn unwrap_or(self, default: T) -> T
    where
        T: ~const Drop + ~const Destruct,
    {
        match self {
            Some(x) => x,
            None => default,
        }
    }
assert_eq!(Some("car").unwrap_or("bike"), "car");
assert_eq!(None.unwrap_or("bike"), "bike");


```

```rust
// 由expect(msg)的定義可以看出是直接取值，或是在None傳回自訂的error msg
pub const fn expect(self, msg: &str) -> T {
        match self {
            Some(val) => val,
            None => expect_failed(msg),
        }
    }

const fn expect_failed(msg: &str) -> ! {
    panic_str(msg)
}
```

```rust
fn main() {
    let x: i8 = 5;
    let y: Option<i8> = Some(5);
    let z: Option<i8> = Some(6);
    // 使用y前，要先取Some中的值
    println!("sum x+y={}", x + y.expect("not a number"));  //10
    // 直接取z的值，為None時直接panic!
    println!("sum x+z={}", x + z.unwrap());  //11   
}
```

```rust
fn main() {
    let a = Some("a");
    let b: Option<&str> = None;
    assert_eq!(a.expect("a is none"), "a");
    //匹配到None會引起panic，列印的錯誤是expect的參數資訊
    // assert_eq!(b.expect("b is none"), "b is none");

    assert_eq!(a.unwrap(), "a"); //如果a是None，則會引起panic
    assert_eq!(b.unwrap_or("b"), "b"); //匹配到None時返回指定值
    let k = 10;
    // 與unwrap_or類似，只不過參數是FnOnce() -> T
    assert_eq!(Some(4).unwrap_or_else(|| 2 * k), 4);
    assert_eq!(None.unwrap_or_else(|| 2 * k), 20);
}
```

## 組合運算元：map

```rust
pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Some(x) => Some(f(x)),    // 將x包裝成Some(f(x))
            None => None,    // None仍然為None
        }
    }
```

有時我們會覺得每次處理`Option`都需要先提取，然後再做相應計算這樣的操作比較麻煩。可用`map`和`and_then`這兩種方法。

map()這個組合運算元可用於 Some -> Some 和 None -> None 這樣的簡單對映。多個不同的 map() 調用可以串起來，這樣更加靈活。

其中map方法和unwrap一樣，也是一系列方法，包括[map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)、[map\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or)和[map\_or\_else](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or\_else)。map會執行參數中閉包的規則，然後將結果再封為Option並返回。

```rust
fn main() {
    let some_str = Some("Hello!");    // Option<&str>
    let some_str_len = some_str.map(|s| s.len()); // Option<&str> -> Option<i32>
    assert_eq!(some_str_len, Some(6));
}
```

## 組合運算元：and\_then

```rust
pub fn and_then<U, F: FnOnce(T) -> Option<U>>(self, f: F) -> Option<U> {
        match self {
            Some(x) => f(x),
            None => None,
        }
    }
```

但是，如果參數本身返回的結果就是Option的話，處理起來就比較麻煩，因為每執行一次map都會多封裝一層，最後的結果有可能是Some(Some(Some(...)))這樣N層Some的巢狀。這時，我們就可以用[and\_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and\_then)來處理了。

```rust
fn main() {
    assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
}

fn sq(x: u32) -> Option<u32> { 
    Some(x * x) 
}
```

## ? - 故障時返回Err物件

如果你調用的函數和你正在寫的函數都返回 `Option` 類型，如果你調用的函數返回 `None`，你的函數也返回 `None`，這時，代碼可以用問號 ？ 操作符簡化。

```rust
fn foo() -> Option<i32> {
    None
}

fn bar() -> Option<String>{
    foo()?;
    Some(String::from("hello world"))
}

fn main(){
    println!("{:?}", bar()); // None
}
```

## 使用unwrap和?解包Option

如果我們unwrap的Option的值是`None`，那麼程式就會`panic!`。

```rust
fn next_birthday(current_age: Option<u8>) -> Option<String> {
	// If `current_age` is `None`, this returns `None`.
	// If `current_age` is `Some`, the inner `u8` gets assigned to `next_age` after 1 is added to it
    let next_age: u8 = current_age?;
    Some(format!("Next year I will be {}", next_age + 1))
}

fn main() {
  let s = next_birthday(None);
  match s {
      Some(a) => println!("{:#?}", a),
      None => println!("No next birthday")
  }
}
```
