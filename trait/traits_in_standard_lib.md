# 標準庫常用trait

## 標準庫常用traits

### Display, Debug

```rust
// std::fmt::Display
pub trait Display {
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error>;
} // std::fmt::Debug
pub trait Debug {
    fn fmt(&self, f: &mut Formatter) -> Result<(), Error>;
}
```

它們的主要用處就是用在類似println！這樣的地方。

只有實現了`Display` trait的類型，才能用`{}`格式控制列印出來；只有實現了`Debug` trait的類型，才能用`{:?}{#?}`格式控制列印出來。

Display假定了這個類型可以用utf-8格式的字串表示，它是準備給最終用戶看的，並不是所有類型都應該或者能夠實現這個trait。這個trait的fmt應該如何格式化字串，完全取決於程式師自己，編譯器不提供自動derive的功能。

標準庫中還有一個常用trait叫作`std::string::ToString`，對於所有實現了Display trait的類型，都自動實現了這個ToString trait。它包含了一個方法`to_string（&self）->String`。任何一個實現了Display trait的類型，我們都可以對它調用`to_string()`方法格式化出一個字串。

Debug則是主要為了調試使用，建議所有的作為API的“公開”類型都應該實現這個trait，以方便調試。它列印出來的字串不是以“美觀易讀”為標準，**編譯器提供了自動derive的功能**。

### PartialOrd/Ord/PartialEq/Eq

因為浮點數中存在Nan，因此浮點數滿足偏序非而全序(Nan不滿足三一律，IEEE754規定)，所以浮點數無法排序。

因此，Rust設計了兩個trait來描述這樣的狀態:一個是`std::cmp::PartialOrd`，表示“偏序”，一個是`std::cmp::Ord`，表示“全序”。

```rust
// 偏序 trait
pub trait PartialOrd<Rhs: ?Sized = Self>: PartialEq<Rhs> {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
    fn lt(&self, other: &Rhs) -> bool {}
    fn le(&self, other: &Rhs) -> bool {}
    fn gt(&self, other: &Rhs) -> bool {}
    fn ge(&self, other: &Rhs) -> bool {}
}
// 全序trait
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;
}
```

partial\_cmp函數的返回數值型別是`Option<Ordering>`。只有Ord trait裡面的cmp函數才能返回一個確定的Ordering。f32和f64類型都只實現了PartialOrd，而沒有實現Ord。

這個設計是優點，而不是缺點，它讓我們盡可能地在更早的階段發現錯誤，而不是留到執行時再去debug。

```rust
fn main() {
    let int_vec = [1_i32, 2, 3];
    let biggest_int = int_vec.iter().max();
    // 編譯錯誤，浮點數不滿足全序，無法排序
    let float_vec = [1.0_f32, 2.0, 3.0];
    // let biggest_float = float_vec.iter().max();
}
```

Rust中的PartialOrd trait實際上就是C++20中即將加入的three-waycomparison運算子<=>。同理，PartialEq和Eq兩個trait也就可以理解了，它們的作用是比較相等關係，與排序關係非常類似。

### Sized

```rust
#[lang = "sized"]
#[rustc_on_unimplemented = "`{Self}` does not have a constant size known at
compile-time"]
#[fundamental] // for Default, for example, which requires that `[T]: !Default` be evaluatable
pub trait Sized {
    // Empty.
}
```

這個trait定義在`std::marker`模組中，它沒有任何的成員方法。它有#\[lang="sized"]屬性，說明它與普通trait不同，編譯器對它有特殊的處理。

用戶也不能針對自己的類型impl這個trait。<mark style="background-color:red;">**一個類型是否滿足Sized約束是完全由編譯器推導的，用戶無權指定**</mark>。

在C/C++這一類的語言中，大部分變數、參數、返回值都應該是編譯階段固定大小的。**在Rust中，但凡編譯階段能確定大小的類型，都滿足Sized約束。**

有些類型在編譯期無法確定大小。

* 一個 slice的`[T]`的size是未知的，因為在編譯期不知道到底會有多少個T存在。
* 一個trait的size是未知的，因為不知道實現這個trait的結構是什麼。

確定類型別的大小(size)對於能夠在堆疊(stack)上為實例分配足夠的空間是十分重要的。確定大小型別(sized type)可以通過傳值(by value)或者傳引用(by reference)的方式來傳遞。

如果一個類型的大小不能在編譯期確定，那麼它就被稱為不確定大小型別(unsized type)或者DST，即動態大小型別(Dynamically-Sized Type)。因為不確定大小類型(unsized type)不能存放在堆疊上，所以它們只能通過傳引用(by reference)的方式來傳遞。

<mark style="background-color:red;">把unsized的類型放到指標或者Box裡面，就變成了sized了</mark>，可通過指標找到源頭，然後順著源頭找到其他的資料。

```rust
use std::mem::size_of;

fn main() {
    // primitives
    assert_eq!(4, size_of::<i32>());
    assert_eq!(8, size_of::<f64>());

    // tuples
    assert_eq!(8, size_of::<(i32, i32)>());

    // arrays
    assert_eq!(0, size_of::<[i32; 0]>());
    assert_eq!(12, size_of::<[i32; 3]>());

    struct Point {
        x: i32,
        y: i32,
    }

    // structs
    assert_eq!(8, size_of::<Point>());

    // enums
    assert_eq!(8, size_of::<Option<i32>>());

    // get pointer width, will be
    // 4 bytes wide on 32-bit targets or
    // 8 bytes wide on 64-bit targets
    const WIDTH: usize = size_of::<&()>();

    // pointers to sized types are 1 width
    assert_eq!(WIDTH, size_of::<&i32>());
    assert_eq!(WIDTH, size_of::<&mut i32>());
    assert_eq!(WIDTH, size_of::<Box<i32>>());
    assert_eq!(WIDTH, size_of::<fn(i32) -> i32>());

    const DOUBLE_WIDTH: usize = 2 * WIDTH;

    // unsized struct
    struct Unsized {
        unsized_field: [i32],
    }

    // pointers to unsized types are 2 widths
    assert_eq!(DOUBLE_WIDTH, size_of::<&str>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&[i32]>()); // slice
    assert_eq!(DOUBLE_WIDTH, size_of::<&dyn ToString>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<Box<dyn ToString>>()); // trait object
    assert_eq!(DOUBLE_WIDTH, size_of::<&Unsized>()); // user-defined unsized type

    // unsized types
    // size_of::<str>(); // compile error
    // size_of::<[i32]>(); // compile error
    // size_of::<dyn ToString>(); // compile error
    // size_of::<Unsized>(); // compile error
}
```

* 在Rust中，指向陣列的動態大小檢視(dynamically sized views)被稱為切片(slice)。例如，一個`&str`是一個"字串切片(string slice)" ，一個`&[i32]`是一個"`i32`切片"。最常見的切片是字串切片`&str`和陣列切片`&[T]`。
* 切片(slice)是雙寬度(double-width)的，因為他們儲存了一個指向陣列的指標和陣列中元素的數量。
* trait物件指標是雙寬度(double-width)的，因為他們儲存了一個指向資料的指標和一個指向vtale的指標。
* 不確定大小(unsized) 結構體指標是雙寬度的，因為他們儲存了一個指向結構體資料的指標和結構體的大小(size)。
* 不確定大小(unsized) 結構體只能擁有有1個不確定大小(unsized)欄位(field)而且它必須是結構體裡的最後一個欄位(field)。

### ?Sized trait

`?Sized`就表示UnSized(不確定類型大小)類型。

`fn foo<T:?Sized>(){}`現在可以接受`UnSized`的資料類型了。

* `?Sized`可以是明顯的“可選大小(optionally sized)”或者“可能大小(maybe sized)”，將其新增到型別引數的約束(bound)上，允許該型別是確定大小(sized)或者不確定大小(unsized)。
* `?Sized`是一個非常特殊的用法，其他的trait bound，都是用來縮小數據的類型範圍的，<mark style="color:red;">這個是用來擴大類型範圍的</mark>。
* `?Sized`是Rust中惟一的寬松約束。

為什麼這很重要？當我們處理泛型引數的時候並且那個型別隱藏在指標背後，我們幾乎總是想要選擇退出預設的Sized約束來讓我們的函式在其將要接受的引數型別上更加自由。而且，如果我們沒有選擇退出預設的Sized約束，我們將最終得到一些令人驚訝和迷惑的編譯錯誤資訊。

<mark style="background-color:red;">Traits預設是?Sized</mark>。trait物件本質上不確定大小(unsized)的，因為任何大小的任意型別都能實現一個trait，因此，如果是`Trait:?Sized`，我們只能為dyn Trait實現trait。

```rust
trait Trait where Self: ?Sized {}
```

### Default

Rust裡面並沒有C++裡面的“構造函數”的概念。它只提供了類似C語言的各種複合類型各自的初始化語法。主要原因在於，**相比普通函數，構造函數本身並沒有提供什麼額外的抽象能力。所以Rust裡面推薦使用普通的靜態函數作為類型的“構造器”**。

如String類型的構造方法非常多如下：

```rust
fn new() -> String
fn with_capacity(capacity: usize) -> String
fn from_utf8(vec: Vec<u8>) -> Result<String, FromUtf8Error>
fn from_utf8_lossy<'a>(v: &'a [u8]) -> Cow<'a, str>
fn from_utf16(v: &[u16]) -> Result<String, FromUtf16Error>
fn from_utf16_lossy(v: &[u16]) -> String
unsafe fn from_raw_parts(buf: *mut u8, length: usize, capacity: usize) -> String
unsafe fn from_utf8_unchecked(bytes: Vec<u8>) -> String
```

這些方法接受的參數各異，錯誤處理方式也各異，強行將它們統一到同名字的構造函數重載中不是什麼好主意。

不過，對於那種無參數、無錯誤處理的簡單情況，標準庫中提供了Default trait來做這個統一抽象。

```rust
trait Default {
    fn default() -> Self;
}
```

它只包含一個“靜態函數”default()返回Self類型。標準庫中很多類型都實現了這個trait，<mark style="color:red;">它相當於提供了一個類型的預設值</mark>。

在Rust中，單詞new並不是一個關鍵字。所以我們可以看到，很多類型中都使用了new作為函數名，用於命名那種最常用的創建新物件的情況。因為這些new函數差別甚大，所以並沒有一個trait來對這些new函數做一個統一抽象。
