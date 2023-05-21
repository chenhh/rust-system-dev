# 所有權與移動(ownership and move)

## 簡介

<mark style="color:red;">所有權是 Rust 用來達成其最大的目標「記憶體安全」的方法</mark>。 它有幾個不同的概念：

* 所有權（ownership）。
* 借用（borrowing）\[變數]，及其相關功能「參照」（references）\[類型]。
* 生命週期 （lifetime），借用的進階概念。

變數的所有權 (ownership) ，在 Rust 中每個變數都有其所屬的範圍 (scope) ，在變數的有效的範圍中，可以選擇將變數「借 (borrow)」給其它的 scope ，也可以將所有權整個轉移 (move) 出去，送給別人。

* 當然，送出去(move)的東西如果別人不還你的話是拿不回來。
* 但借出去(borrow)的就只是暫時的給別人使用而已。

![rust語義樹](<../.gitbook/assets/rust-min (1).jpg>)

語義樹，主要是想表達下面幾層意思：

1. 所有權是有好多個概念系統性組成的一個整體概念。
2. let繫結，繫結了什麼？變數 + 作用域 + 資料（記憶體）。
3. move、lifetime、RAII都是和作用域相關的，所以想理解它們就先要理解作用域。

## 所有權(ownership)

所有權，顧名思義，至少應該包含兩個物件：“所有者”和“所有物”。<mark style="color:red;">在Rust中，“所有者”就是變數，“所有物”是資料，抽象來說，就是指某一片記憶體。</mark><mark style="color:red;background-color:red;">let關鍵字，允許你繫結“所有者”和“所有物”</mark>。

“所有權”代表著以下意義：

* 每個值在Rust中都有一個變數來管理它，這個變數就是這個值、這塊記憶體的所有者；
* <mark style="color:red;">每個值在一個時間點上只有一個所有者(不能同時有多個所有者，但可以借用)</mark>；
* 當變數所在的作用域結束的時候，變數以及它代表的值將會被銷毀(類似RAII)。

**這代表當綁定離開有效範圍，Rust 就會釋放所綁定的資源**。

### 變數作用域(scope)

```rust
{                      // 變數s在這裡無效, 它尚未聲明
    let s = "hello";   // 從此處起，s是有效的
    // 使用 s
}                      // 此作用域已結束，s不再有效
```

```rust
fn main() {
    // 從此處起，變數s是有效的
    let mut s = String::from("hello");
    // 因為對s有寫入的操作，因此必須宣告為mut
    s.push_str(" world");
    println!("{}", s);
}
// 此作用域已結束，
// s 不再有效
```

* 當我們聲明一個變數s，並用String類型對它進行初始化的時候，這個變數s就成了這個字串的“所有者” let將s繫結至String，但這個所有權，是有範圍限制的，這個範圍就是<mark style="color:red;">作用域（scope</mark>），準確來說，應該叫擁有域（owner scope）。
* 如果我們希望修改這個變數，可以使用mut修飾s，然後調用String類型的成員方法來實現。
* 當main函數結束的時候，s將會被解構，它管理的記憶體（不論是堆積上的，還是堆疊上的）則會被釋放。
* <mark style="color:blue;">我們一般把變數從出生到死亡的整個階段，叫作一個變數的“生命週期”（lifetime）</mark>。比如這個例子中的區域變數s，它的生命週期就是從let語句開始，到main函數結束。
* <mark style="color:red;">當變量離開作用域，Rust 為我們調用一個特殊的解構函數</mark><mark style="color:red;">`drop`</mark><mark style="color:red;">。Rust 在結尾的 } 處自動調用 drop</mark>。

```rust
fn main() {
    let s = String::from("hello");
    let s1 = s;
    // error, borrow of moved value: `s`
    // 因為s對String的所有權已經轉移給s1，
    // s的生命週期結束, 不可再被使用
    // 解法：改用borrow，let s1 = &s;
    // 解法：改用clone, let s1 = s.clone();
    println!("{}", s);
}

// 上面以mir可檢視作用域
scope 1 {
    let _1  std::string::string; // s in scope 1
    scope 2 {
        let _2 std::string::string; // s1 in scope 2
    }
}
```

在Rust裡面，<mark style="background-color:orange;">不可以做“設定運算子重載”</mark>，<mark style="color:red;">若需要“深複製” (deep copy)（指的是對於複雜結構中所有元素的複製，而不是單純以共享指標指向該結構），必須手工調用clone方法。</mark>這個clone方法來自於std:clone::Clone這個trait。clone方法裡面的行為是可以自訂的。

### 所有權與函數

將值傳遞給函數在語義上與給變量賦值相似。<mark style="color:red;">向函數傳遞值可能會移動或者復制(依傳入類型是否實作copy trait決定行為)</mark>，就像賦值語句一樣。

<mark style="color:blue;">在每一個函數中都獲取所有權並接著返回所有權有些囉嗦。如果我們想要函數使用一個值但不獲取所有權可使用引用(借用)</mark>。

```rust
fn main() {
    let s = String::from("hello"); // s 進入作用域

    takes_ownership(s); // s 的值移動到函數裡 ...
                        // ... 所以到這裡不再有效
                        // hello

    let x = 5; // x 進入作用域

    makes_copy(x); // x 應該移動函數裡，
                   // 但 i32 是 Copy 的，所以在後面可繼續使用 x
} // 這裡, x 先移出了作用域，然後是 s。但因為 s 的值已被移走，
  // 所以不會有特殊操作

fn takes_ownership(some_string: String) {
    // some_string 進入作用域
    // 因為String沒有實作copy trait，為move操作
    println!("{}", some_string);    
} // 這裡，some_string 移出作用域並調用 `drop` 方法。佔用的內存被釋放

fn makes_copy(some_integer: i32) {
    // some_integer 進入作用域
    // 因為i32有實作copy trait,所以為copy操作
    println!("{}", some_integer);
} // 這裡，some_integer 移出作用域。不會有特殊
```

### 返回值與作用域

返回值也可以轉移所有權。

<mark style="color:red;">變量的所有權總是遵循相同的模式：將值賦給另一個變量時移動它。當持有堆積中資料值的變量離開作用域時，其值將通過 drop 被清理掉，除非資料被移動為另一個變量所有</mark>。

```rust
fn main() {
    let s1 = gives_ownership(); // gives_ownership 將返回值
                                // 移給 s1

    let s2 = String::from("hello"); // s2 進入作用域

    let s3 = takes_and_gives_back(s2); // s2 被移動到
                                       // takes_and_gives_back 中,
                                       // 它也將返回值移給 s3
} // 這裡, s3 移出作用域並被丟棄。s2 也移出作用域，但已被移走，
  // 所以什麼也不會發生。s1 移出作用域並被丟棄

fn gives_ownership() -> String {
    // gives_ownership 將返回值移動給
    // 調用它的函數

    let some_string = String::from("hello"); // some_string 進入作用域.

    some_string // 返回 some_string 並移出給調用的函數
}

// takes_and_gives_back 將傳入字串並返回該值
fn takes_and_gives_back(a_string: String) -> String {
    // a_string 進入作用域

    a_string // 返回 a_string 並移出給調用的函數
}
```

## 移動語意(move)

一個變數可以把它擁有的值轉移給另外一個變數，稱為“所有權轉移”。設定陳述式、函式呼叫、函數返回等，都有可能導致所有權轉移。

<mark style="color:red;">**Rust中所有權轉移的重要特點是，它是所有類型的預設語義**</mark>。Rust中的變數綁定操作，預設是move語義，執行了新的變數綁定後，原來的變數就不能再被使用！一定要記住！

```rust
let s1 = String::from("hello");
let s2 = s1;
```

![將值 "hello" 綁定給 s1 的 String 在記憶體中的表現形式](../.gitbook/assets/string\_ctor-min%20\(1\).png)

![s1對hello的所有權移動至s2](../.gitbook/assets/string\_move-min.png)

```rust
fn create(){
    let s = String::from("hello");
    return s; // 所有權轉移,從函數內部移動到外部
}
fn consume(s: String) {
    // 所有權轉移,從函數外部移動到內部
    println!("{}", s);
}
fn main() {
    let s = create();
    consume(s);
}
```

所有權轉移的步驟分解如下。

1. main函式呼叫create函數。
2. 在調用create函數的時候創建了字串，在堆疊上(s)和堆積上(hello)都分配有記憶體。區域變數s是這些記憶體的所有者。
3. create函數返回的時候，需要將區域變數s移動到函數外面，這個過程就是簡單地按位元組複製memcpy。
4. 同理，在調用consume函數的時候，需要將main函數中的區域變數轉移到consume函數，這個過程也是簡單地按位元組複製memcpy。
5. 當consume函數結束的時候，它並沒有把內部的區域變數再轉移出來，這種情況下，consume內部區域變數的生命週期就該結束了。這個區域變數s生命週期結束的時候，會自動釋放它所擁有的記憶體，因此字串也就被釋放了。

C++的做法就不一樣了，它允許賦值構造函數(copy constructor)、設定運算子(assignment operator)重載，因此在出現“構造”或者“賦值”操作的時候，有可能表達的是完全不同的含義，這取決於程式設計師如何實現重載。C++的這個設計具有巨大的靈活性，但是不恰當的實現也可能造成記憶體不安全。

在C++裡面，`std::vector<int>v1=v2`；是複製語義，而Rust裡面的`let v1:Vec<i32>=v2`；是移動語義。如果要在Rust裡面實現複製語義，需要顯式寫出函式呼叫`let v1:Vec<i32>=v2.clone()`；

“語義”不代表最終的執行效率。“語義”只是規定了什麼樣的程式碼是編譯器可以接受的，以及它執行後的效果可以用怎樣的思維模型去理解。編譯器有權在不改變語義的情況下做任何有利於執行效率的優化。語義和優化是兩個階段的事情。我們可以把移動語義想像成執行了一個memcpy，但真實的組譯代碼未必如此。

編譯器可以提前在當前調用堆疊中把大物件的空間分配好，然後把這個物件的指標傳遞給子函數，由子函數執行這個變數的初始化。這樣就避免了大物件的複製工作，參數傳遞只是一個指標而已。這麼做是完全滿足移動語義要求的，而且編譯器還有權利做更多類似的優化。

## 複製語意(copy)

* 此處的複製指的是隱式的copy，而非顯式的clone。
* move可以類比成作業系統中的剪下(或移動)操作，移動後，原始的資料就不可再被使用。

<mark style="background-color:blue;">預設的move語義是Rust的一個重要設計</mark>，但是任何時候需要複製都去調用clone函數會顯得非常煩瑣。對於一些簡單類型，比如整數、bool，讓它們在賦值的時候默認採用複製操作會讓語言更簡單。

<mark style="background-color:orange;">基本類型有實作copy trait, // =運算子會使用copy而非move</mark>。

```rust
fn main() {
    let v1: isize = 0;
    // 基本類型有實作copy trait,
    // =運算子會使用copy而非move
    // 所有權沒有被轉移，
    // v2和v1指向的是不同記憶體地址
    let v2 = v1;
    println!("{}", v1); // 0
    // 因為是複製，所以指向相異記憶體地址
    println!("{:p}, {:p}", &v1, &v2);
}
```

因為在Rust中有一部分特殊的類型，其變數綁定操作是copy語義。**所謂的copy語義，是指在執行變數綁定操作的時候，v2是對v1所屬資料的一份複製。v1所管理的這塊記憶體依然存在，並未失效，而v2是新開闢了一塊記憶體，它的內容是從v1管理的記憶體中複製而來的**。

和手動調用clone方法效果一樣，`let v2=v1；`等效於`let v2=v1.clone();`。使用檔案系統來打比方。

* copy語義就像“複製、粘貼”操作。操作完成後，原來的資料依然存在，而新的資料是原來資料的複製品。
* move語義就像“剪切、粘貼”操作。操作完成後，原來的資料就不存在了，被移動到了新的地方。

這兩個操作本身是一樣的，都是簡單的記憶體複製，區別在於複製完以後，原先那個變數的生命週期是否結束。

Rust中，在普通變數綁定、函數傳參、模式匹配等場景下，<mark style="color:red;">凡是實現了</mark><mark style="color:red;">`std::marker::Copy`</mark> <mark style="color:red;">trait的類型，都會執行copy語義</mark>。**基本類型，比如整數、浮點數、字元、bool等，都實現了Copy trait，因此具備copy語義。對於自訂類型，預設是沒有實現Copy trait的，但是我們可以手動實現**。

```rust
struct Foo {
    data: i32,
}
// Copy繼承了Clone，我們要實現Copy trait必須同時實現Clone trait。
impl Clone for Foo {
    fn clone(&self) -> Foo {
        Foo { data: self.data }
    }
}
impl Copy for Foo {}
fn main() {
    let v1 = Foo { data: 0 };
    // 實現copy trait後，=運算子為複製語言
    let v2 = v1;
    let v3 = &v1;
    println!("{:?}", v1.data);
    // v3指向v1相同的地址，而v2是copy
    println!("{:p}, {:p}, {:p}", &v1, &v2, v3);
}
```

編譯通過。現在Foo類型也擁有了複製語義。在執行變數綁定、函數參數傳遞的時候，原來的變數不會失效，而是會新開闢一塊記憶體，將原來的資料複製過來。

絕大部分情況下，實現Copy trait和Clone trait是一個非常機械化的、重複性的工作，clone方法的函數體要對每個成員調用一下clone方法。<mark style="color:red;">**Rust提供了一個編譯器擴展derive attribute，來幫我們寫這些程式碼，其使用方式為**</mark><mark style="color:red;">**`#[derive（Copy，Clone）]`**</mark><mark style="color:red;">**。只要一個類型的所有成員都具有Clone trait，我們就可以使用這種方法來讓編譯器幫我們實現Clone trait了**</mark>。

```rust
// 由編譯器自動實現Foo的copy與clone trait
#[derive(Copy, Clone)]
struct Foo {
    data: i32,
}
fn main() {
    let v1 = Foo { data: 0 };
    let v2 = v1;
    println!("{:?}", v1.data);
}
```

## Box類型

[std::boxed::Box](https://doc.rust-lang.org/std/boxed/struct.Box.html)

Box類型是Rust中一種常用的指標類型。它代表“擁有所有權的指標”，類似於C++裡面的`unique_ptr`（嚴格來說，`unique_ptr<T>`更像`Option<Box<T>>`）。

在 Rust 中，所有值預設都是堆疊分配的。通過創建 `Box<T>`，可以把值裝箱（boxed）來 使它在堆積上分配。箱子（box，即 `Box<T>` 類型的實例）是一個智慧指標，指向堆積分配的 T 類型的值。當箱子離開作用域時，它的解構函數會被調用，內部的對象會被銷毀，堆積上分配的記憶體也會被釋放。

被裝箱的值可以使用 \* 運算符進行解引用；這會移除掉一層裝箱。

```rust
use std::mem;

#[derive(Debug, Clone, Copy)]
struct Point {
    x: f64,
    y: f64,
}

struct Rectangle {
    p1: Point,
    p2: Point,
}

fn origin() -> Point {
    Point { x: 0.0, y: 0.0 }
}

fn boxed_origin() -> Box<Point> {
    // 在堆上分配這個點（point），並返回一個指向它的指標
    Box::new(Point { x: 0.0, y: 0.0 })
}

fn main() {
    // （所有的型別標注都不是必需的）
    // 堆疊分配的變數
    let point: Point = origin();
    let rectangle: Rectangle = Rectangle {
        p1: origin(),
        p2: Point { x: 3.0, y: 4.0 },
    };

    // 堆積分配的 rectangle（矩形）
    let boxed_rectangle: Box<Rectangle> = Box::new(Rectangle {
        p1: origin(),
        p2: origin(),
    });

    // 函式的輸出可以裝箱
    let boxed_point: Box<Point> = Box::new(origin());

    // 兩層裝箱
    let box_in_a_box: Box<Box<Point>> = Box::new(boxed_origin());

    println!(
        "Point occupies {} bytes in the stack",
        mem::size_of_val(&point)    // 16
    );
    println!(
        "Rectangle occupies {} bytes in the stack",
        mem::size_of_val(&rectangle)    // 32
    );

    // box 的寬度就是指標寬度
    println!(
        "Boxed point occupies {} bytes in the stack",
        mem::size_of_val(&boxed_point) // 8
    );
    println!(
        "Boxed rectangle occupies {} bytes in the stack",
        mem::size_of_val(&boxed_rectangle)  // 8
    );
    println!(
        "Boxed box occupies {} bytes in the stack",
        mem::size_of_val(&box_in_a_box) // 8
    );

    // 將包含在 `boxed_point` 中的資料複製到 `unboxed_point`
    let unboxed_point: Point = *boxed_point;
    println!(
        "Unboxed point occupies {} bytes in the stack",
        mem::size_of_val(&unboxed_point)    // 16
    );
}
```

```rust
struct T {
    value: i32,
}
fn main() {
    // 使用new method建構Box物件, p為box pointer
    let p = Box::new(T { value: 1 });
    println!("{}", p.value);
}
```

**Box類型永遠執行的是move語義，不能是copy語義。Rust中的copy語義就是淺複製。對於Box這樣的類型而言，淺複製必然會造成二次釋放問題**。

對於Rust裡面的所有變數，在使用前一定要合理初始化，否則會出現編譯錯誤。對於`Box<T>/&T/&mut T`這樣的類型，合理初始化意味著它一定指向了某個具體的物件，不可能是空。如果使用者確實需要“可能為空的”指標，必須使用類型`Option<Box<T>>`。

Rust裡面還有一個保留關鍵字box（注意是小寫）。它可以用於把變數“裝箱”到堆上。目前這個語法依然是unstable狀態，需要打開feature gate才能使用。

## clone與copy trait

Rust中的[std::marker::Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html)是一個特殊的trait，它給類型提供了“複製”語義。在Rust標準庫裡面，還有一個跟它很相近的trait，叫作[std::clone::Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html) (copy繼承自clone trait)。很容易把這兩者混淆。

```rust
//  std::marker::Copy
pub trait Copy: Clone { }
```

* Copy內部沒有方法，Clone內部有兩個方法。
* **Copy trait是給編譯器用的**，告訴編譯器這個類型預設採用copy語義，而不是move語義。
* **Clone trait是給程式設計師用的**，我們必須手動呼叫clone方法，它才能發揮作用。
* Copy trait不是想實現就能實現的，它對類型是有要求的，有些類型不可能impl Copy。而Clone trait則沒有什麼前提條件，任何類型都可以實現（unsized類型除外，因為無法使用unsized類型作為返回值）。
* Copy trait規定了這個類型在執行變數綁定、函數參數傳遞、函數返回等場景下的操作方式。即這個類型在這種場景下，必然執行的是“簡單記憶體複製”操作，這是由編譯器保證的，程式師無法控制。
* Clone trait裡面的clone方法究竟會執行什麼操作，則是取決於程式師自己寫的邏輯。一般情況下，clone方法應該執行一個“深複製”操作，但這不是強制性的。
* 如果你確實不需要Clone trait執行其他自訂操作（絕大多數情況都是這樣），編譯器提供了一個工具，我們可以在一個類型上添加`#[derive（Clone）]`，來讓編譯器幫我們自動生成那些重複的程式碼。_編譯器自動生成的clone方法非常機械，就是依次調用每個成員的clone方法_。
* Rust語言規定了在`T：Copy` (類型必須有實現copy)的情況下，Clone trait代表的含義。即：當某變數`t：T`符合`T：Copy`時，它調用`t.clone（）`方法的含義必須等同於“簡單記憶體複製”。也就是說，clone的行為必須等同於`let x=std::ptr::read(&t);`，也等同於`let x=t;`。當`T:Copy`時，我們不要在Clone trait裡面亂寫自己的邏輯。所以，當我們需要指定一個類型是Copy的時候，最好使用`#[derive（Copy，Clone）]`方式，避免手動實現Clone導致錯誤。

### copy的含意

Copy的全名是`std::marker::Copy`。請大家注意，[std::marker](https://doc.rust-lang.org/std/marker/index.html)模組裡面所有的trait都是特殊的trait。目前穩定的有四個，它們是Copy、Send、Sized、Sync、(目前多了一個unpin)。

**它們的特殊之處在於:它們是跟編譯器密切綁定的，impl這些trait對編譯器的行為有重要影響**。在編譯器眼裡，它們與其他的trait不一樣。**這幾個trait內部都沒有方法，它們的唯一任務是給類型打一個“標記”，表明它符合某種約定——這些約定會影響編譯器的靜態檢查以及程式碼生成**。

Copy這個trait在編譯器的眼裡代表的是什麼意思呢？簡單點總結就是，如果一個類型impl了Copy trait，意味著任何時候，我們都可以通過簡單的記憶體複製（在C語言裡按位元組複製memcpy）實現該類型的複製，並且不會產生任何記憶體安全問題。<mark style="color:red;">**一旦一個類型實現了Copy trait，那麼它在變數綁定、函數參數傳遞、函數返回值傳遞等場景下，都是copy語義，而不再是預設的move語義**</mark>。

下面用最簡單的設定陳述式`x=y`來說明move語義和copy語義的根本區別。

* move語義是“剪切、粘貼”操作，變數`y`把所有權遞交給了`x`之後，`y`就徹底失效了，後面繼續使用`y`就會出編譯錯誤。
* 而copy語義是“複製、粘貼”操作，變數`y`把所有權遞交給了`x`之後，它自己還留了一個副本，在這句設定陳述式之後，`x`和`y`依然都可以繼續使用。

**在Rust裡，move語義和copy語義具體執行的操作，是不允許由程式師自訂的，這是它和C++的巨大區別**。這裡沒有賦值構造函數或者設定運算子重載。**move語義或者copy語義都是執行的memcpy，無法更改，這個過程中絕對不存在其他副作用**。

當然，這裡一直談的是“語義”，而沒有涉及編譯器優化。從語義的角度，我們要講清楚，什麼樣的程式碼在編譯器看來是合法的，什麼樣的程式碼是非法的。如果考慮後端優化，在許多情況下，不必要的記憶體複製實際上已經徹底優化掉了，大家不必擔心執行效率的問題。也沒有必要每次都把move或者copy操作與具體的組譯代碼聯繫起來，因為場景不同，優化結果不同，生成的程式碼也是不同的。大家只需記住的是語義。

### Copy的實現條件

並不是所有的類型都可以實現Copy trait。

<mark style="color:red;">**Rust規定，對於自訂類型，只有所有成員都實現了Copy trait，這個類型才有資格實現Copy trait**</mark>。

<mark style="color:red;">常見的數字（整數、浮點數）類型、bool類型、共用借用指標&，都是具有Copy屬性的類型</mark>。而Box、Vec、可寫借用指標\&mut等類型都是不具備Copy屬性的類型。

對於陣列類型，如果它內部的元素類型是Copy，那麼這個陣列也是Copy類型。對於元組tuple類型，如果它的每一個元素都是Copy類型，那麼這個tuple也是Copy類型。

struct和enum類型不會自動實現Copy trait。只有當struct和enum內部的每個元素都是Copy類型時，編譯器才允許我們針對此類型實現Copy trait。

比如下面這個類型，雖然它的成員是Copy類型，但它本身不是Copy類型：

```rust
#[derive(Copy, Clone)]
struct T(i32);
fn main() {
    let t1 = T(1);
    // 必須實作copy trait才是copy語意，否則為move
    let t2 = t1;
    println!("{} {}", t1.0, t2.0);
}
```

### Clone的含意

Clone的全名是std::clone::Clone。它的完整聲明如下：

```rust
pub trait Clone: Sized {
    fn clone(&self) -> Self;
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone()
    }
}
```

它有兩個關聯方法，分別是clone\_from和clone，clone\_from是有預設實現的，依賴於clone方法的實現。clone方法沒有預設實現，需要手動實現。

clone方法一般用於“基於語義的複製”操作。所以，它做什麼事情，跟具體類型的作用息息相關。比如，對於Box類型，clone執行的是“深複製”；而對於Rc類型，clone做的事情就是把引用計數值加1。

雖然Rust中的clone方法一般是用來執行複製操作的，但是如果在自訂的clone函數中做點別的什麼工作，編譯器也沒辦法禁止。你可以根據需要在clone函數中編寫任意的邏輯。

<mark style="color:red;">**但是有一條規則需要注意：對於實現了copy的類型，它的clone方法應該跟copy語義相容，等同於按位元組複製**</mark>。

## 自動derive

絕大多數情況下，實現Copy Clone這樣的trait都是一個重複而無聊的工作。因此，Rust提供了一個attribute，讓我們可以利用編譯器自動生成這部分程式碼。

```rust
#[derive(Copy, Clone)]
struct MyStruct(i32);
```

這裡的derive會讓編譯器幫我們自動生成impl Copy和impl Clone這樣的程式碼。自動生成的clone方法，會依次調用每個成員的clone方法。通過derive方式自動實現Copy和手工實現Copy有微小的區別。

當類型具有泛型參數的時候，比如`struct MyStruct<T>{}`，通過derive自動生成的程式碼會自動添加一個T：Copy的約束。目前，只有一部分固定的特殊trait可以通過derive來自動實現。將來Rust會允許自訂的derive行為，讓我們自己的trait也可以通過derive的方式自動實現。

## 參考資料

* [\[知乎\] Rust所有權語義模型](https://zhuanlan.zhihu.com/p/27571264?group\_id=862978524611497984)
