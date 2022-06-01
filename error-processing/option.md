# Option

## 簡介

Option是一種枚舉類型，主要包括兩種值：`Some(T)`和`None`，Rust也是靠它來避免空指標異常的。

## 取值的方法

一般用兩種方法取值，一種是expect，另一種是unwrap系列的方法。

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

有時我們會覺得每次處理Option都需要先提取，然後再做相應計算這樣的操作比較麻煩。可用map和and\_then這兩種方法。

其中map方法和unwrap一樣，也是一系列方法，包括map、map\_or和map\_or\_else。map會執行參數中閉包的規則，然後將結果再封為Option並返回。

```rust
fn main() {
    let some_str = Some("Hello!");
    let some_str_len = some_str.map(|s| s.len());
    assert_eq!(some_str_len, Some(6));
}
```

但是，如果參數本身返回的結果就是Option的話，處理起來就比較麻煩，因為每執行一次map都會多封裝一層，最後的結果有可能是Some(Some(Some(...)))這樣N多層Some的巢狀。這時，我們就可以用and\_then來處理了。

```rust
fn main() {
    assert_eq!(Some(2).and_then(sq).and_then(sq), Some(16));
}

fn sq(x: u32) -> Option<u32> { 
    Some(x * x) 
}
```

## ? - 故障時返回Err物件

如果你調用的函數和你正在寫的函數都返回 Option 類型，如果你調用的函數返回 None，你的函數也返回 None，這時，代碼可以用問號 ？ 操作符簡化。

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

如果我們unwrap的Option的值是None，那麼程式就會panic!。

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
