# 模式解構(pattern destructure)

## 簡介

Rust中模式解構功能設計得非常美觀，它的原則是：構造和解構遵循類似的語法，我們怎麼把一個資料結構組合起來的，我們就怎麼把它拆解開來。

注意這裡的“Destructure”和“Destructor”是完全不同的兩個單詞，代表完全不同的含義。

* “Destructure”的意思是把原來的結構肢解為單獨的、局部的、原始的部分；
* “Destructor”是指“解構器”，是一個與“構造器”相對應的概念，是在物件被銷毀的時候調用的。
* Rust的“模式解構”功能不僅出現在match語句中，還可以出現在let、if-let、while-let、函式呼叫、閉包調用等情景中；
* Rust的“模式解構”功能可以應用於各種資料類型，包括但不限於tuple、struct、enum等，暫時在穩定版中不支援slice的模式匹配；
* Rust的“模式解構”功能要求“無遺漏”的分析（exhaustive case analysis），確保不會因為不小心而漏掉某些情況；
* Rust的“模式解構”與Rust的核心所有權管理功能完全相容。



```rust
let tuple = (1_i32, false, 3f32);    // 建構tuple
let (head, center, tail) = tuple;    // 解構tuple至各變數
```

```rust
/* 我們首先構造了一個T2類型的變數x，
 * 它內部又嵌套包含了其他的結構體。
 * 實際上，我們完全可以一次性解構多個層次，
 * 直接把這個物件內部深處的元素拆解出來
*/
// struct tuple
struct T1(i32, char);
// struct
struct T2 {
    item1: T1,
    item2: bool,
}
fn main() {
    // 建構T2 structure
    let x = T2 {
        item1: T1(0, 'A'),
        item2: false,
    };
    // 解構x至T2 struct對應的變數中
    let T2 {
        item1: T1(value1, value2),
        item2: value3,
    } = x;
    println!("{} {} {}", value1, value2, value3);
}
```

Rust的“模式解構”功能不僅出現在let語句中，還可以出現在match、if let、while let、函式呼叫、閉包調用等情景中。而match具有功能最強大的模式匹配。

## match

```rust
enum Direction {
    East,
    West,
    South,
    North,
}
fn print(x: Direction) {
    // 因此match列舉了所有enum的狀態，
    // 所以不必加上_的例外處理
    match x {
        Direction::East => {
            println!("East");
        }
        Direction::West => {
            println!("West");
        }
        Direction::South => {
            println!("South");
        }
        Direction::North => {
            println!("North");
        }
    }
}
fn main() {
    let x = Direction::East;
    print(x);
}
```

Rust要求match需要對所有情況做完整的、無遺漏的匹配，如果漏掉了某些情況，是不能編譯通過的。exhaustive意思是無遺漏的、窮盡的、徹底的、全面的。exhaustive是Rust模式匹配的重要特點。

```rust
fn print(x: Direction) {
    match x {
        Direction::East => {
            println!("East");
        }
        Direction::West => {
            println!("West");
        }
        Direction::South => {
            println!("South");
        }
        // 因此沒有列出Direction::North，
        // 可用_捕抓未列出的所有狀態
        _ => {
            println!("Other");
        }
    }
}
```

正因為如此，在多個專案之間有依賴關係的時候，在上游的一個庫中對enum增加成員，是一個破壞相容性的改動。因為增加成員後，很可能會導致下游的使用者match語句編譯不過。為解決這個問題，Rust提供了一個叫作non\_exhaustive的功能（目前還沒有穩定）。

```rust
#[non_exhaustive]
pub enum Error {
    NotFound,
    PermissionDenied,
    ConnectionRefused,
}
```

上游庫作者可以用一個叫作“non\_exhaustive”的attribute來標記一個enum或者struct，這樣在另外一個項目中使用這個類型的時候，無論如何都沒辦法在match運算式中通過列舉所有的成員實現完整匹配，必須使用底線才能完成編譯。這樣，以後上游庫裡面為這個類型添加新成員的時候，就不會導致下游專案中的編譯錯誤了因為它已經存在一個預設分支匹配其他情況。

### match用法小結

```rust
#[derive(Debug)]
enum OpenJS {
    Nodejs,
    React
}
enum Language {
    Go,
    Rust,
    JavaScript(OpenJS),
}

fn get_url_by_language (language: Language) -> String {
    match language {
        // 僅需要返回一個值，可以不使用大括號。
        Language::Go => String::from("https://golang.org/"),
        // 需要在分支中執行多行代碼，可以使用大括號。
        Language::Rust => {
            println!("We are learning Rust.");
            String::from("https://www.rust-lang.org/")
        },
        // 想對匹配的模式綁定一個值，可以修改枚舉的一個成員來存放數據，
        // 這種模式稱為綁定值的模式。
        Language::JavaScript(value) => {
            println!("Openjs value {:?}!", value);
            String::from("https://openjsf.org/")
        },
    }
}

fn main() {
    print!("{}\n", get_url_by_language(Language::JavaScript(OpenJS::Nodejs)));
    print!("{}\n", get_url_by_language(Language::JavaScript(OpenJS::React)));
    print!("{}\n", get_url_by_language(Language::Go));
    print!("{}\n", get_url_by_language(Language::Rust));
}
```

### match可做為表達式

```rust
enum Direction {
    East,
    West,
    South,
    North,
}
fn direction_to_int(x: Direction) -> i32 {
    // 參數x傳入後，直接match傳回傳(注意match後無分號)
    match x {
        Direction::East => 10,
        Direction::West => 20,
        Direction::South => 30,
        Direction::North => 40,
    }
}
fn main() {
    let x = Direction::East;
    let s = direction_to_int(x);
    println!("{}", s);  // 10
}
```

### 匹配 Option 與 Some(value)

Option 是 Rust 系統定義的一個枚舉類型，它有兩個變量：None 表示失敗、Some(value) 是元組結構體，封裝了一個范型類型的值 value。

```rust
fn something(num: Option<i32>) -> Option<i32> {
    match num {
        // None => None 是必須寫的，否則會報 pattern None not covered 錯誤，
        // 編譯階段就不會通過的。
        None => None,
        Some(value) => Some(value + 1),
    }
}
fn main() {
    let five = Some(5);
    let six = something(five);
    let none = something(None);

    println!("{:?} {:?}", six, none);
}
```

### 底線(underline)可表示省略單個元素

除了在match中表示未捕捉到的所有狀態之外，**底線還能用在模式匹配的各種地方，用來表示一個預留位置，雖然匹配到了但是忽略它的值的情況**：

```rust
struct P(f32, f32, f32);
fn calc(arg: P) -> f32 {
    // 模式解構, 匹配 tuple struct,
    // 但是忽略第二個成員的值
    let P(x, _, y) = arg;
    x * x + y * y
}
fn main() {
    let t = P(1.0, 2.0, 3.0);
    println!("{}", calc(t));  // 10
}
```

上例可在參數傳入函數時，直接解構且忽略變數：

```rust
struct P(f32, f32, f32);
// 參數類型是 P,參數本身是一個模式,解構之後,變數x、y分別綁定了第一個和第三個成員
fn calc(P(x, _, y): P) -> f32 {
    x * x + y * y
}
fn main() {
    let t = P(1.0, 2.0, 3.0);
    println!("{}", calc(t)); // 10
}
```

底線更像是一個“關鍵字”，而不是普通的“識別字”（identifier），把它當成普通識別字使用是會有問題的,，編譯器並不會把單獨的底線當成一個正常的變數名處理。

**如果把底線後面跟上字母、數位或者底線，那麼它就可以成為一個正常的識別字了**。比如，連續兩個底線\_\_，就是一個合法的、正常的“識別字”。

`let _=x`；和`let _y=x`；具有不一樣的意義。這一點在後面的“解構函數”部分還會繼續強調。

* 如果變數`x`是非Copy類型，`let _=x`；的意思是“**忽略綁定**”，此時會直接調用`x`的解構函數，我們不能在後面使用底線\_讀取這個變數的內容；
* 而`let _y=x`；的意思是“**所有權轉移**”，`_y`是一個正常的變數名，`x`的所有權轉移到了`_y`上，`_y`在後面可以繼續使用。

底線在Rust裡面用處很多，比如：在match運算式中表示“其他分支”，在模式中作為預留位置，還可以在類型中做預留位置，在整數和小數字面量中做連接子，等等。

### ..表示省略多個元素

除了底線可以在模式中作為“預留位置”，還有兩個點`..`也可以在模式中作為“預留位置”使用。底線`_`表示省略一個元素，兩個點可以表示省略多個元素。

```rust
fn main() {
    let x = (1, 2, 3, 4);
    // pattern destucture,
    // 使用底線必須匹配a之後的元素個數
    let (a, _, _, _) = x;
    println!("{}", a); // 1
    
    // 可用..表示省略a之後所有元素
    let (a, ..) = x;
    println!("{}", a); // 1
    // ..也可表示省略給定區間中的所有元素
    let (a, .., b) = x;
    println!("{}, {}", a, b); // 1, 4
}
```

Match可以匹配值，且可以使用或運算子|來匹配多個條件：

```rust
fn category(x: i32) {
    match x {
        -1 | 1 => println!("true"),
        0 => println!("false"),
        _ => println!("error"),
    }
}
fn main() {
    let x = 1;
    category(x);    // true
}
```

match可用`..`表示前閉後開區間，或是`..=`表示閉區間：

```rust
fn category(x: char) {
    match x {
        'a'..='z' => println!("lowercase"),
        'A'..='Z' => println!("uppercase"),
        _ => println!("something else"),
    }
}
fn main() {
    let x = 'c';
    category(x); // lowercase
}
```

## 匹配看守(match guard)

可以使用if作為“匹配看守”（match guards）。當匹配成功且符合if條件，才執行後面的語句。

```rust
enum OptionalInt {
    Value(i32),
    Missing,
}
fn main() {
    let x = OptionalInt::Value(5);
    match x {
        OptionalInt::Value(i) if i > 5 => println!("Got an int bigger than five!"),
        OptionalInt::Value(..) => println!("Got an int!"),
        OptionalInt::Missing => println!("No such luck."),
    }
}
```

在對變數的“值”進行匹配的時候，編譯器依然會保證“完整無遺漏”檢查。但是這個檢查目前做得並不是很完美，某些情況下會發生誤報的情況，因為畢竟編譯器內部並沒有一個完整的數學解算功能。

```rust
fn main() {
    let x = 10;
    
    match x {
        i if i > 5 => println!("bigger than five"),
        i if i <= 5 => println!("small or equal to five"),
        _ => unreachable!(),    // 必須加上此分支，否則會出現編譯錯誤
    }
}
```

<mark style="background-color:blue;">編譯器會保證match的所有分支合起來一定覆蓋了目標的所有可能取值範圍，但是並不會保證各個分支是否會有重疊的情況（畢竟編譯器不想做成一個完整的數學解算器）</mark>。如果不同分支覆蓋範圍出現了重疊，各個分支之間的先後順序就有影響。

```rust
fn intersect(arg: i32) {
    match arg {
        // 如果我們進行匹配的值同時符合好幾條分支，
        // 那麼總會執行第一條匹配成功的分支，忽略其他分支
        i if i < 0 => println!("case 1"),
        i if i < 10 => println!("case 2"),
        i if i * i < 1000 => println!("case 3"),
        _ => println!("default case"),
    }
}
fn main() {
    let x = -1;
    intersect(x);   // case 1
}
```

## match的變數綁定

我們可以使用`@`符號綁定變數。`@`符號前面是新聲明的變數，後面是需要匹配的模式。

```rust
fn main() {
    let x = 1;
    match x {
        // 以變數以代表匹配到的值
        e @ 1..=5 => println!("got a range element {}", e),
        _ => println!("anything"),
    }
}
```

如果在使用`@`的同時使用|，需要保證在每個條件上都綁定這個名字：

```rust
fn main() {
    let x = 5;
    match x {
        e @ 1..=5 | e @ 8..=10 => println!("got a range element {}", e),
        _ => println!("anything"),
    }
}
```

## ref關鍵字

如果我們需要綁定的是被匹配物件的引用，則可以使用ref關鍵字。之所以在某些時候需要使用ref，是因為模式匹配的時候有可能發生變數的所有權轉移，**使用ref就是為了避免出現所有權轉移**。

```rust
fn main() {
    let x = 5_i32;
    match x {
        // 此時 r 的類型是 `&i32`
        ref r => println!("Got a reference to {}", r),
    }
}
```

### ref關鍵字和引用符號&的關係

* <mark style="background-color:red;">**ref是“模式”的一部分，它只能出現在賦值號左邊**</mark>；
* 而&符號是借用運算子，是運算式的一部分，它只能出現在賦值號右邊。
* ref關鍵字是“模式”的一部分，不能修飾賦值號右邊的值。`let x=ref 5_i32；`這樣的寫法是錯誤的語法。

為了搞清楚這些變數綁定的分別是什麼類型，我們可以把變數的類型資訊列印出來看看。有兩種方案：

* 利用編譯器的錯誤資訊來幫我們理解；
* 利用標準庫裡面的intrinsic函數列印。

```rust
// 目前只能在nightly channel執行
#![feature(core_intrinsics)]

fn print_type_name<T>(_arg: &T) {
    unsafe {
        println!("{}", std::intrinsics::type_name::<T>());
    }
}

fn main() {
    let x = 5_i32;      // i32
    let x = &5_i32;     // &i32
    print_type_name(&x);
    let ref x = 5_i32;  // &32
    print_type_name(&x);
    let ref x = &5_i32; // &&32
    print_type_name(&x);
}
```

## mut關鍵字

mut關鍵字也可以用於模式綁定中。mut關鍵字和ref關鍵字一樣，是“模式”的一部分。**Rust中，所有的變數綁定預設都是“不可更改”的。只有使用了mut修飾的變數綁定才有修改資料的能力**。

```rust
fn main() {
    let mut v = vec![1i32, 2, 3];
    v = vec![4i32, 5, 6]; // 重新綁定到新的Vec
    //v = vec![1.0f32, 2, 3]; // 類型不匹配,不能重新綁定
}
```

重新綁定與前面提到的“變數遮蔽”（shadowing）是完全不同的作用機制。“重新綁定”要求變數本身有mut修飾，並且不能改變這個變數的類型。“變數遮蔽”要求必須重新聲明一個新的變數，這個新變數與老變數之間的類型可以毫無關係。

Rust在“可變性”方面，預設為不可修改。與C++的設計剛好相反。C++預設為可修改，使用const關鍵字修飾的才變成不可修改。

mut關鍵字不僅可以在模式用於修飾變數綁定，還能修飾指標（引用），這裡是很容易搞混的地方。**mut修飾變數綁定，與\&mut型引用，是完全不同的意義**。

```rust
let mut x: &mut i32;
//  ^1     ^2
```

以上兩處的mut含義是不同的。

* 第1處mut，**代表這個變數x本身可變，因此它能夠重新綁定到另外一個變數上去**，具體到這個示例來說，就是指標的指向可以變化。
* 第2處mut，修飾的是指標，**代表這個指標對於所指向的記憶體具有修改能力**，因此我們可以用\*x=1；這樣的語句，改變它所指向的記憶體的值。

## if let, while let關鍵字

Rust不僅能在match運算式中執行“模式解構”，在let語句中，也可以應用同樣的模式。Rust還提供了if-let語法糖。它的語法為`if let PATTERN=EXPRESSION{BODY}`。後面可以跟一個可選的else分支。

```rust
// 原始寫法，從Some中取出x傳入函數
// 對於 None 值我們不希望做任何操作。
match optVal {
    Some(x) => {
        doSomethingWith(x);
    }
    _ => {}
}

// 簡化寫法一
// 首先判斷它一定是 Some(_)
if optVal.is_some() { 
    // 然後取出內部的資料
    let x = optVal.unwrap(); 
    doSomethingWith(x);
}

// 簡化寫法二，使用if let
// 如果optVal回傳值為Some時，以x解構Some並執行doSomethingWith(x)
// 否則不做任何事
if let Some(x) = optVal {
    doSomethingWith(x);
}
```

這其實是一個簡單的語法糖，其背後執行的程式碼與match運算式相比，並無效率上的差別。<mark style="background-color:red;">它跟match的區別是：match一定要完整匹配，if-let只匹配感興趣的某個特定的分支(不處理None)，這種情況下的寫法比match簡單點。</mark>同理，while-let與if-let一樣，提供了在while語句中使用“模式解構”的能力，此處就不再舉例。

使用 if let 意味著編寫更少代碼，更少的縮進和更少的樣板代碼。<mark style="color:blue;">然而，這樣會失去 match 強制要求的窮盡性檢查。</mark>match 和 if let 之間的選擇依賴特定的環境以及增加簡潔度和失去窮盡性檢查的權衡取捨。
