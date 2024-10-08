# Option

## 簡介

* [https://doc.rust-lang.org/std/option/enum.Option.html](https://doc.rust-lang.org/std/option/enum.Option.html)
* [https://rustwiki.org/zh-CN/std/option/index.html#enums](https://rustwiki.org/zh-CN/std/option/index.html#enums)

Option是一種枚舉類型，主要包括兩種值：`Some(T)`和`None`，強制使用者判斷是否存在空值。

```rust
enum Option<T> {
    Some(T),    // Some表示有值, tuple struct
    None,       // None表示沒有值
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

## unwrap()方法族

[unwrap](https://doc.rust-lang.org/std/option/enum.Option.html#method.unwrap)如果為 Some，則返回所包含的值，否則時直接panic。

```rust
// 由unwrap()的定義可以看出是直接取值，或在None時直接panic
// 回傳值會造成所有權轉移
pub const fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic("called `Option::unwrap()` on a `None` value"),
        }
    }
```

unwrap會消費Option變數，後面不可再使用。且有所有權的限制，只有原始變數可使用unwrap，其引用(可變與不可變)均不可呼叫之。

```rust
fn main() {
    let a = Some(Box::new(5));
    let av = a.unwrap(); // 所有權已轉移，無法再使用a
    println!("val={}", *av);

    let b = Some(Box::new(6));
    let c = &b;
    // cannot move out of `*c` which is behind a shared reference
    //   let d = *c.unwrap();

    // 先clone再unwrap()就OK
    let e = c.clone().unwrap();
}
```

### unwrap\_or方法

unwrap\_or(default)取值失敗時，回傳default。

generic bounds前的tilde(\~)符號表示不穩定的實作。

```rust
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

### unwrap\_or\_default()方法

如果為 Some，則返回所包含的值，否則，如果None，則返回回傳類型的預設值。

```rust
// 如果 Some，則返回所包含的值，否則，如果None，則返回該類型的預設值。
pub const fn unwrap_or_default(self) -> T
    where
        T: ~const Default,
    {
        match self {
            Some(x) => x,
            None => Default::default(),
        }
    }
let good_year_from_input = "1909";
let bad_year_from_input = "190blarg";
let good_year = good_year_from_input.parse().ok().unwrap_or_default();
let bad_year = bad_year_from_input.parse().ok().unwrap_or_default();

assert_eq!(1909, good_year);
assert_eq!(0, bad_year);
```

### unwrap\_or\_else()方法

```rust
// 返回包含的 Some 值或從閉包中計算得出
// FnOnce：表示捕獲方式為通過值（T）的閉包。
pub const fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: ~const FnOnce() -> T,
        F: ~const Drop + ~const Destruct,
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
let k = 10;
assert_eq!(Some(4).unwrap_or_else(|| 2 * k), 4);
assert_eq!(None.unwrap_or_else(|| 2 * k), 20);
```

### expect(msg)方法

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

### unwrap\_unchecked()方法

```rust
// 返回包含 self 值的包含的 Some 值，而不檢查該值是否不是 None。
pub const unsafe fn unwrap_unchecked(self) -> T {
        debug_assert!(self.is_some());
        match self {
            Some(val) => val,
            // SAFETY: the safety contract must be upheld by the caller.
            None => unsafe { hint::unreachable_unchecked() },
        }
    }
    
let x: Option<&str> = None;
assert_eq!(unsafe { x.unwrap_unchecked() }, "air"); // 未定義的行為！s
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

## take方法

take方法可以從option中取值，而原始的變數為None(原始變數仍有效，只是值變為None)，注意變數必須為可變。

```rust
pub const fn take(&mut self) -> Option<T> {
        // FIXME replace `mem::replace` by `mem::take` when the latter is const ready
        mem::replace(self, None)
    }
```

```rust
fn main() {
    let mut a = Some(Box::new(5));
    let mut b = &mut a;
    let c = &mut b;
    let d = c.take();

    println!("{:?}", c); //None
    println!("{:?}", d); //Some(5)
    println!("{:?}", a); // None
}
```

## 組合運算元：map族

[map(f)](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)如果變數為None時，直接傳回None，否則將變數unwrap後，再呼叫函數回傳計算值`Some(f(x))`。

```rust
//FnOnce：表示捕獲方式為通過值（T）的閉包。
pub const fn map<U, F>(self, f: F) -> Option<U>{
        match self {
            Some(x) => Some(f(x)),    // 將x包裝成Some(f(x))
            None => None,             // None仍然為None
        }
    }

let maybe_some_string = Some(String::from("Hello, World!"));
// `Option::map` takes self *by value*, consuming `maybe_some_string`
let maybe_some_len = maybe_some_string.map(|s| s.len()); // 得到Some(s.len())

assert_eq!(maybe_some_len, Some(13));
```

### map\_or方法

[map\_or(default, f)](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or)如果變數為None時，直接傳回給定預設值default值，否則將變數unwrap後，再呼叫函數回傳計算值Some(f(x))。

```rust
// 返回提供的預設結果 (如果None)，或將函數應用於包含的值 (如果Some)。
pub const fn map_or<U, F>(self, default: U, f: F) -> U
    where
        F: ~const FnOnce(T) -> U,
        F: ~const Drop + ~const Destruct,
        U: ~const Drop + ~const Destruct,
    {
        match self {
            Some(t) => f(t), // 將t包裝成f(t)
            None => default, // None傳default之值
        }
    }

let x = Some("foo");
assert_eq!(x.map_or(42, |v| v.len()), 3);

let x: Option<&str> = None;
assert_eq!(x.map_or(42, |v| v.len()), 42);
```

### map\_or\_else方法

[map\_or\_else(default\_func, f)](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or\_else)如果變數為None時，直接傳回給定預設函數default，否則將變數unwrap後，再呼叫函數回傳計算值Some(f(x))。

```rust
// 計算 default 函數的結果 (如果None)，或將不同的函數應用於包含的值 (如果Some)。
pub const fn map_or_else<U, D, F>(self, default: D, f: F) -> U
    where
        D: ~const FnOnce() -> U,
        D: ~const Drop + ~const Destruct,
        F: ~const FnOnce(T) -> U,
        F: ~const Drop + ~const Destruct,
    {
        match self {
            Some(t) => f(t),   // 將t包裝成f(t)
            None => default(), // None傳回default()函數
        }
    }

let k = 21;
let x = Some("foo");
assert_eq!(x.map_or_else(|| 2 * k, |v| v.len()), 3);

let x: Option<&str> = None;
assert_eq!(x.map_or_else(|| 2 * k, |v| v.len()), 42);
```

有時我們會覺得每次處理`Option`都需要先提取，然後再做相應計算這樣的操作比較麻煩。可用`map`和`and_then`這兩種方法。

`map()`這個組合運算元可用於 Some -> Some 和 None -> None 這樣的簡單對映。多個不同的 `map()` 調用可以串起來，這樣更加靈活。

其中map方法和unwrap一樣，也是一系列方法，包括[map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)、[map\_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or)和[map\_or\_else](https://doc.rust-lang.org/std/option/enum.Option.html#method.map\_or\_else)。map會執行參數中閉包的規則，然後將結果再封為Option並返回。

```rust
fn main() {
    let some_str = Some("Hello!");    // Option<&str>
    let some_str_len = some_str.map(|s| s.len()); // Option<&str> -> Option<i32>
    assert_eq!(some_str_len, Some(6));
}
```

## 組合運算元：and\_then

如果變數為None時，直接傳回None，否則將變數unwrap後，再呼叫函數回傳計算值`f(x)`。

```rust
//FnOnce：表示捕獲方式為通過值（T）的閉包。
 pub const fn and_then<U, F>(self, f: F) -> Option<U>
    where
        F: ~const FnOnce(T) -> Option<U>,
        F: ~const Drop + ~const Destruct,
    {
        match self {
            Some(x) => f(x),   // x傳回函數f(x)值
            None => None,     //None時仍為None
        }
    }
```

如果參數本身返回的結果就是`Option`的話，處理起來就比較麻煩，因為每執行一次map都會多封裝一層，最後的結果有可能是Some(Some(Some(...)))這樣N層Some的巢狀。這時，我們就可以用[and\_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and\_then)來處理了。

```rust
fn main() {
    assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
}

fn sq(x: u32) -> Option<u32> { 
    Some(x * x) 
}
```

## ? - 故障時返回Err物件

[The question mark operator](https://doc.rust-lang.org/reference/expressions/operator-expr.html)

問號算子無法被重載。

如果呼叫的函數和正在寫的函數都返回 `Option` 類型，如果呼叫的函數返回 `None`，你的函數也返回 `None`，這時，代碼可以用問號 ？ 操作符簡化。如果是Result類型，則回傳Error。

```rust
fn foo() -> Option<i32> {
    None
}

fn bar() -> Option<String>{
    foo()?; // None時直接return, 不會執行下一步
    // 等價寫法，None必須加上return表示提早返回
    // match foo() {
    //     Some(T) => Some(T),
    //     None => return None,
    // };
    
    Some(String::from("hello world"))
}

fn main(){
    println!("{:?}", bar()); // None
}
```

### 運算子?可以用於連續的函數呼叫中

```rust
/* 
這個函數返回 Option<char> 因為它可能會在這個位置找到一個字元，也可能沒有字元。
獲取 text 字串 slice 作為參數並呼叫其 lines 方法，這會返回一個字串中每一行的迭代器。
因為函數希望檢查第一行，所以呼叫了迭代器 next 來獲取迭代器中第一個值。
如果 text 是空字串，next 呼叫會返回 None，此時我們可以使用 ? 來停止並從 last_char_of_first_line 返回 None。
如果 text 不是空字串，next 會返回一個包含 text 中第一行的字串 slice 的 Some 值。
? 會提取這個字串 slice，然後可以在字串 slice 上呼叫 chars 來獲取字元的迭代器。
我們感興趣的是第一行的最後一個字元，所以可以呼叫 last 來返回迭代器的最後一項。
*/
fn last_char_of_first_line(text: &str) -> Option<char> {
    text.lines().next()?.chars().last()
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
