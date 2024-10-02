# 類型轉換

## 常用轉換

Rust中類型非常嚴格，即使i32與i64都無法直接計算，必須先轉型。

常用規則如下：

* 數值類型轉換後可以使用(通常是記憶體較短轉為較長的類型)，而浮點數轉整數會無條件捨去，而非四捨五入。
* 字串轉為數值資料，可用`x.trim().parse.unwrap()`或`x.trim().parse::<i32>().unwrap()`。
* 數值轉字串類型，使用`x.to_string()`或者`format!`。
* char轉為數值類型，使用`x.to_digit(10).unwrap()`，其中10表示10進位。
* 字串轉為bytes類型，使用`x.as_bytes()`。
* bytes轉為字串類型，使用`String::from_utf8(x.to_vec()).unwrap()`。

```rust
fn main() {
    // 數值資料轉換
    let x1: i32 = 10;
    let x2: f32 = 20.5;
    let result = x1 + x2 as i32;
    println!("{result}"); // 30而不是31
    
    // 字串轉數值
    let x1: String = "10".to_string();
    let x2: f32 = 20.5;
    let result = x1.trim().parse::<f32>().unwrap() + x2;
    println!("{result}"); // 30.5
    
    // 數值轉字串
    let x1: f32 = 20.5;
    let result = x1.to_string() + "元";
    let result2 = format!("{}元", x1.to_string());
    println!("{result}, {result2}"); // 20.5元, 20.5元
    
    // char轉數值
    let x1 = '9';
    let result = x1.to_digit(10).unwrap(); // 10進位
    println!("{result}"); // 9
    let x1 = 'f';
    let result = x1.to_digit(16).unwrap(); //16進位
    println!("{result}"); // 15
    
    // 字串轉bytes
    let x1 = "中文";
    println!("{:?}", x1.as_bytes()); // [228, 184, 173, 230, 150, 135]
    
    // bytes轉string
    let x2 = String::from_utf8(x1.as_bytes().to_vec()).unwrap();
    println!("{x2}"); // 中文
    
} 

```

## 強制類型轉換

在一些特定場景中，類型會被隱式地強制轉換。這種轉換通常導致類型被 “弱化”，主要針對指標和生命週期。主要目的是讓 Rust 適用於更多的場景，並且基本上是無害的。

如下幾種類型之間允許進行強制轉換：

* 傳遞性：當`T_1` 可以強制轉換為`T_2` 且`T_2` 可以強制轉換為`T_3` 時，`T_1` 就可以強制轉換為 `T_3`。
* 指標弱化：
  * `&mut T`轉換成`&T`。
  * `*mut T` 轉換為 `*const T`。
  * `&T` 轉換為 `*const T`。
  * `&mut T` 轉換為 `*mut T`。
* Unsize：如果 `T` 實現了 `CoerceUnsized<U>`，那麼 `T` 可以強制轉換為 `U`。
* 強制解引用：如果`T` 可以解引用為 `U`（比如 `T: Deref<Target=U>`），那麼 `&T` 類型的表示式 `&x` 可以強制轉換為 `&U` 類型的 `&*x`。
* 所有的指標類型（包括 Box 和 Rc 這些智慧指針）都實現了`CoerceUnsized<Pointer> for Pointer where T: Unsize`。

強制轉換會在 “強制轉換位置” 處發生。每一個顯式聲明了類型的位置都會引起到該類型的強制轉換。<mark style="color:red;">但如果必須進行類型推斷，則不會發生類型轉換</mark>。

表示式 `e` 到類型 `U` 的強制轉換位置包括：

* let 表示式，靜態變數或者常數：`let x: U = e`。
* 函數的參數：`takes_a_U(e)`。
* 函數返回值：`fn foo() -> U {e}`。
* 結構體初始化：`Foo { some_u: e }`。
* 陣列初始化：`let x: [U; 10] = [e, ...]`。
* 元組初始化：`let x: (U, ..) = (e, ..)`。
* 程式碼塊中的最後一個表示式：l`et x: U = { ..; e }`。

注意，<mark style="color:red;">在匹配 trait 的時候不會發生強制類型轉換</mark>（receiver 除外，具體見下）。也就是說，如果為 U 實現了一個 trait，T 可以強制轉換為 U，並不能認為 T 也實現了這個 trait。

## 顯式類型轉換

所有的強制類型轉換都可以通過顯式轉換的方式主動觸發。但有一些場景只適用於顯式轉換。強制類型轉換很普遍而且通常無害，但是顯式類型轉換是一種 “真正的轉換 “，它的應用就很稀少了，而且有潛在的危險。

顯式轉換必須通過關鍵字 as 主動地觸發：`expr as Type`。

```
// 不顯示類型轉換產生的溢出警告。
#![allow(overflowing_literals)]
```

<mark style="color:red;">Rust對不同類型之間的轉換控制得非常嚴格。必須使用as關鍵字，否則會出現編譯錯誤</mark>。

Rust設計者希望在發生類型轉換的時候不是偷偷摸摸進行的，而是顯式地標記出來，防止隱藏的bug。雖然在許多時候會讓程式碼顯得不那麼精簡，但這也算是一種合理的折中。

```rust
fn main() {
    let var1: i8 = 41;
    // let var2 : i16 = var1;     // compile error, 不可直接轉型
    let var2 : i16 = var1 as i16; // 必須使用as關鍵字(顯示轉換)
    
    // 當把任何類型轉換為無符號類型 T 時，會不斷加上或減去 (std::T::MAX + 1)
    // 直到值位於新類型 T 的範圍內。
    // 1000 已經在 u16 的範圍內
    println!("1000 as a u16 is: {}", 1000 as u16); // 1000

    // 1000 - 256 - 256 - 256 = 232
    // 事實上的處理方式是：從最低有效位（LSB，least significant bits）開始保留
    // 8 位，然後剩餘位置，直到最高有效位（MSB，most significant bit）都被拋棄。
    // 譯注：MSB 就是二進位的最高位元，LSB 就是二進位的最低位元，按日常書寫習慣就是
    // 最左邊一位和最右邊一位。
    println!("1000 as a u8 is : {}", 1000 as u8);  //232
    // -1 + 256 = 255
    println!("  -1 as a u8 is : {}", (-1i8) as u8); // 255

    // 對正數，這就和取模一樣。
    println!("1000 mod 256 is : {}", 1000 % 256);

    // 當轉換到有符號類型時，（位元操作的）結果就和 “先轉換到對應的無符號類型，
    // 如果 MSB 是 1，則該值為負” 是一樣的。

    // 當然如果數值已經在目標類型的範圍內，就直接把它放進去。
    println!(" 128 as a i16 is: {}", 128 as i16);
    // 128 轉成 u8 還是 128，但轉到 i8 相當於給 128 取八位元的二進位補數，其值是：
    println!(" 128 as a i8 is : {}", 128 as i8);

    // 重複之前的例子
    // 1000 as u8 -> 232
    println!("1000 as a u8 is : {}", 1000 as u8);
    // 232 的二進位補數是 -24
    println!(" 232 as a i8 is : {}", 232 as i8);
}
```

<mark style="color:blue;">as關鍵字也不是隨便可以用的，它只允許編譯器認為合理的類型轉換</mark>。任意類型轉換是不允許的，有些時候，甚至需要連續寫多個as才能轉成功，比如\&i32類型就不能直接轉換為\*mut i32類型。

```rust
fn main() {
    let a = "some string";
    //let b = a as u32; // compile error, 不合理的轉換
    
    let i = 42;
    // let p = &i as *mut i32; //無法直接轉型
    
    // 先轉為 *const i32,再轉為 *mut i32
    let p = &i as *const i32 as *mut i32;
    
    println!("{:p}", p);
}
```

as關鍵字對於表達式`e as U`，e為表達式，U為轉換的目標類型，合法的轉換型別如下表：

| e的類型                  | 目標類型                  |
| --------------------- | --------------------- |
| integer or float type | integer or float type |
| C-like enum           | Integer type          |
| bool or char          | integer type          |
| u8                    | char                  |
| \*T                   | \*V where V: sized \* |
| \*T where T: Sized    | numeric type          |
| Integer type          | \*V where V: sized    |
| &\[T; n]              | \*const T             |
| Function pointer      | \*V where V: sized    |
| Function pointer      | Integer               |

更複雜的類型轉換，可使用標準庫的From, Into等trait。

[https://learnku.com/docs/nomicon/2018/43-explicit-type-conversion/4726](https://learnku.com/docs/nomicon/2018/43-explicit-type-conversion/4726)

### from trait

From 和 Into 兩個 trait 是內部相關聯的，實際上這是它們實現的一部分。如果我們能夠從類型 B 得到類型 A，那麼很容易相信我們也能把類型 B 轉換為類型 A。

From trait 允許一種類型定義 “怎麼根據另一種類型生成自己”，因此它提供了一種類型轉換的簡單機制。在標准庫中有無數 From 的實現，規定原生類型及其他常見類型的轉換功能。

```rust
// 把 str 轉換成 String
let my_str = "hello";
let my_string = String::from(my_str);
```

為我們自己的類型定義轉換機制：

```rust
use std::convert::From;

#[derive(Debug)]
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let num = Number::from(30);
    println!("My number is {:?}", num);
}
```

### into trait

Into trait 就是把 From trait 倒過來而已。也就是說，如果你為你的類型實現了 From，那麼同時你也就免費獲得了 Into。

使用 Into trait 通常要求指明要轉換到的類型，因為編譯器大多數時候不能推斷它。不過考慮到我們免費獲得了 Into，這點代價不值一提。

```rust
use std::convert::From;

#[derive(Debug)]
struct Number {
    value: i32,
}

impl From<i32> for Number {
    fn from(item: i32) -> Self {
        Number { value: item }
    }
}

fn main() {
    let int = 5;
    // 類型說明不可省略
    let num: Number = int.into();
    println!("My number is {:?}", num);
}
```

TryFrom . TryInto trait

類似於 From 和 Into，TryFrom 和 TryInto 是 類型轉換的通用 trait。不同於 From/Into 的是，TryFrom 和 TryInto trait 用於易出錯的轉換，也正因如此，其返回值是 Result 型。

```rust
use std::convert::TryFrom;
use std::convert::TryInto;

#[derive(Debug, PartialEq)]
struct EvenNumber(i32);

impl TryFrom<i32> for EvenNumber {
    type Error = ();

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value % 2 == 0 {
            Ok(EvenNumber(value))
        } else {
            Err(())
        }
    }
}

fn main() {
    // TryFrom

    assert_eq!(EvenNumber::try_from(8), Ok(EvenNumber(8)));
    assert_eq!(EvenNumber::try_from(5), Err(()));

    // TryInto

    let result: Result<EvenNumber, ()> = 8i32.try_into();
    assert_eq!(result, Ok(EvenNumber(8)));
    let result: Result<EvenNumber, ()> = 5i32.try_into();
    assert_eq!(result, Err(()));
}
```

### ToString, FromStr trait

要把任何類型轉換成 String，只需要實現那個類型的 ToString trait。然而不要直接這麼做，您應該實現fmt::Display trait，它會自動提供 ToString，並且還可以用來列印類型。

```rust
use std::string::ToString;

struct Circle {
    radius: i32
}

impl ToString for Circle {
    fn to_string(&self) -> String {
        format!("Circle of radius {:?}", self.radius)
    }
}

fn main() {
    let circle = Circle { radius: 6 };
    println!("{}", circle.to_string());
}
```

### 解析字串

我們經常需要把字串轉成數字。完成這項工作的標准手段是用 parse 函數。我們得 提供要轉換到的類型，這可以通過不使用類型推斷，或者用 “渦輪魚” 語法（turbo fish，<>）實現。

只要對目標類型實現了 FromStr trait，就可以用 parse 把字串轉換成目標類型。 標准庫中已經給無數種類型實現了 FromStr。如果要轉換到用戶定義類型，只要手動實現 FromStr 就行。

```rust
fn main() {
    let parsed: i32 = "5".parse().unwrap();
    let turbo_parsed = "10".parse::<i32>().unwrap();

    let sum = parsed + turbo_parsed;
    println!{"Sum: {:?}", sum};
}
```
