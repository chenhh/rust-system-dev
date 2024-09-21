# trait語法

在Rust中，trait這一個概念有多個意義。在中文裡，trait可以翻譯為“特徵”“特點”“特性”等。由於這些詞區分度並不明顯，因此不翻譯trait這個詞，以避免歧義。trait中可以包含：函數、常量、類型等。

* trait本身可以攜帶泛型參數；
* trait可以用在泛型參數的約束中；
* trait可以為一組類型impl，也可以單獨為某一個具體類型impl，而且它們可以同時存在；
* trait可以為某個trait impl，而不是為某個具體類型impl；·
* rait可以包含關聯類型，而且還可以包含類型構造器，實現高階類型的某些功能；
* trait可以實現泛型程式碼的靜態分派，也可以通過trait object實現動態分派；
* trait可以不包含任何方法，用於給類型做標籤（marker），以此來描述類型的一些重要特性；
* trait可以包含常量。

## 成員函數(membership function)

trait中可以定義函數，稱為<mark style="color:red;">該trait的成員函數(membership function)</mark>。該函數可直接實作，或等到imp時再實作即可，類似於interface的概念。

```rust
trait Shape {
    fn area(&self) -> f64;    //成員函數，尚未實作
}
```

### self參數

<mark style="background-color:red;">所有的trait中都有一個隱藏的</mark><mark style="background-color:red;">**類型Self（大寫S），代表當前這個實現了此trait的具體類型**</mark><mark style="background-color:red;">。trait中定義的函數，也可以稱作關聯函數（associated function）</mark>。

函數的第一個參數如果是Self相關的類型，且命名為self（小寫s），這個參數可以被稱為“receiver”（接收者）。

Rust中Self（大寫S）和self（小寫s）都是關鍵字，<mark style="color:red;">大寫S的是類型名(實作trait的struct名稱類型)，小寫s的是變數名(綁定實作的struct的實例)</mark>，一定要注意區分。

* <mark style="color:blue;">trait中具有receiver參數的函數，我們稱為“方法”（method）</mark>可以通過變數實例使用小數點來調用。
* <mark style="color:blue;">trait中沒有receiver參數的函數，我們稱為“靜態函數”（static function）</mark>，可以通過類型加雙冒號::的方式來調用。
* **在Rust中，函數和方法沒有本質區別(與變數是否實例化無關?)**。

對於第一個self參數，常見的類型有`self:Self`、`self:&Self`、`self:&mut Self`等類型。對於以上這些類型，Rust提供了一種簡化的寫法，我們可以將參數簡寫為`self`、`&self`、`&mut self`。

<mark style="color:red;">self參數只能用在第一個參數的位置</mark>。請注意“變數self”和“類型Self”的大小寫不同。

```rust
trait T {
    // 參數為self, 類型為Self, copy or move
    fn method1(self: Self);
    // 參數為self, 類型為&Self, immutable borrow
    fn method2(self: &Self);
    // 參數為self, 類型為&mut Self, mutable borrow
    fn method3(self: &mut Self);
    
    fn mymethod(); // static function
}
// 上下兩種寫法等價
trait T {
    fn method1(self);
    fn method2(&self);
    fn method3(&mut self);
    fn mymethod();
}
```

trait中可以包含方法的預設實現。如果這個方法在trait中已經有了方法體，那麼在針對具體類型實現的時候，就可以選擇不用重寫。當然，如果需要針對特殊類型作特殊處理，也可以選擇重新實現來“override”預設的實現方式。

比如，在標準庫中，反覆運算器Iterator這個trait中就包含了十多個方法，但是，其中只有`fn next（&mut self）->Option<Self::Item>`是沒有預設實現的。其他的方法均有其預設實現，在實現反覆運算器的時候只需挑選需要重寫的方法來實現即可。

### 預設方法

```rust
trait Foo {
    fn is_valid(&self) -> bool;
    fn is_invalid(&self) -> bool { !self.is_valid() }
}
```

`is_invalid`是預設方法，實現者可不必實現它，如果選擇實現它，會覆蓋掉它的預設行為。

## 實現trait

我們可以為某些具體類型實現（impl）這個trait。而且要實現所有的成員函數。

假如我們有一個結構體類型Circle，它實現了這個trait，程式碼如下：

```rust
trait Shape {
    fn area(self: &Self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    // Self 類型就是 Circle
    // self 的類型是 &Self,即 &Circle
    fn area(&self) -> f64 {
        // 訪問成員變數,需要用 self.radius
        std::f64::consts::PI * self.radius * self.radius
    }
}
fn main() {
    let c = Circle { radius: 2_f64 };
    // 第一個參數名字是 self,可以使用小數點語法調用
    println!("The area is {}", c.area());
    
    // 內在方法等價寫法
    println!("The area is {}", Circle::area(&c));
}
```

### 類型的內在方法

另外，針對一個類型，我們可以直接對它impl來增加成員方法，無須trait名字（可視為該類型特有的匿名trait）。用這種方式定義的方法叫作這個<mark style="color:red;">類型的“內在方法”（inherent methods）</mark>。

```rust
// 實現Circle類型的匿名trait, 類似class的概念
impl Circle {
    fn get_radius(&self) -> f64 {
        self.radius
    }
}
```

### 為trait實現trait

```rust
trait Shape {
    fn area(&self) -> f64;
}
trait Round {
    fn get_radius(&self) -> f64;
}
struct Circle {
    radius: f64,
}
impl Round for Circle {
    fn get_radius(&self) -> f64 {
        self.radius
    }
}
// 注意這裡是 impl Trait for Trait
// Round不是類型而是trait，因此要加dyn，
// 表示佔用的記憶體為動態
impl Shape for dyn Round {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.get_radius() * self.get_radius()
    }
}
fn main() {
    // Circle實現trait Round, 而trait Round實現Shape
    // 但是Circle不可直接使用Shape的method, 
    // 因為self的類型不同
    let c = Circle { radius: 2f64 };
    // 編譯錯誤
    // c.area();
    let b = Box::new(Circle { radius: 4f64 }) as Box<dyn Round>;
    // 編譯正確
    b.area();
}
```

### 靜態方法

沒有receiver參數的方法（第一個參數不是self參數的方法）稱作“靜態方法”。靜態方法可以通過`Type::FunctionName()`的方式調用。

需要注意的是，即便我們的第一個參數是Self相關類型，只要變數名字不是self，就不能使用小數點的語法調用函數。

```rust
// struct tuple
struct T(i32);
impl T {
    // 這是一個靜態方法，
    // 因為第一個參數的名字不是self
    fn func(this: &Self) {
        println! {"value {}", this.0};
    }
}
fn main() {
    let x = T(42);
    // x.func(); 小數點方式調用是不合法的
    T::func(&x);
}
```

在標準庫中就有一些這樣的例子。Box的一系列方法`Box::into_raw(b:Self)`,`Box::leak(b:Self)`，以及Rc的一系列方法`Rc::try_unwrap(this:Self)`,`Rc::downgrade(this:&Self)`，都是這種情況。

<mark style="background-color:red;">它們的receiver不是self關鍵字，這樣設計的目的是強制使用者用</mark><mark style="background-color:red;">`Rc::downgrade(&obj)`</mark><mark style="background-color:red;">的形式調用，而禁止</mark><mark style="background-color:red;">`obj.downgrade()`</mark><mark style="background-color:red;">形式的調用</mark>。**這樣程式碼表達出來的意思更清晰，不會因為`Rc<T>`裡面的成員方法和T裡面的成員方法重名而造成誤解問題**。

trait中也可以定義靜態函數。下面以標準庫中的`std::default::Default` trait為例，介紹靜態函數的相關用法

```rust
// 上面這個trait中包含了一個default()函數，它是一個無參數的函數，
// 返回的類型是實現該trait的具體類型。Rust中沒有“構造函數”的概念。
// Default trait實際上可以看作一個針對無參數構造函數的統一抽象。
pub trait Default {
    fn default() -> Self;
}

impl<T> Default for Vec<T> {
        fn default() -> Vec<T> {
            Vec::new()
        }
    }
```

跟C++相比，在Rust中，定義靜態函數沒必要使用static關鍵字，因為它把self參數顯式在參數列表中列出來了。

作為對比，C++裡面成員方法預設可以訪問this指標，因此它需要用static關鍵字來標記靜態方法。Rust不採取這個設計，主要原因是self參數的類型變化太多，不同寫法語義差別很大，選擇顯式聲明self參數更方便指定它的類型。

### 關聯類型(associated types)

關聯類型是一個將類型佔位符與 trait 相關聯的方式，這樣 trait 的方法簽名中就可以使用這些佔位符類型。

一個帶有關聯類型的 trait 的例子是標准庫提供的 Iterator trait。它有一個叫做 Item 的關聯類型來替代遍歷的值的類型。

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

關聯類型看起來像一個類似泛型的概念，因為它允許定義一個函數而不指定其可以處理的類型。那麼為什麼要使用關聯類型呢？**Ans: 區別在於使用泛型時，則不得不在每一個實現中標注類型**。通過關聯類型，則無需標注類型，因為不能多次實現這個 trait。

```rust
//關聯類型
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
}
// 泛型
pub trait Iterator<T> {
    // 每次都要為回傳值的T標注類型
    fn next(&mut self) -> Option<T>;
}
```

## 擴展類型方法

我們還可以利用trait給其他的類型添加成員方法，哪怕這個類型不是我們自己寫的。

比如，我們可以為內置類型i32添加一個方法：

```rust
trait Double {
    fn double(&self) -> Self;
}
// 為內建類型實現trait
impl Double for i32 {
    fn double(&self) -> i32 {
        *self * 2
    }
}
fn main() {
    // 可以像成員方法一樣調用
    let x: i32 = 10.double();
    println!("{}", x);
}
```

即使實作的類型不是在當前的專案中聲明的，我們依然可以為它增加一些成員方法。

## 孤兒規則(orphan rule)

但我們也不是隨隨便便就可以這麼做的，Rust對此有一個規定。在聲明trait和impl trait的時候，**Rust規定了一個Coherence Rule（一致性規則）或稱為Orphan Rule（孤兒規則）：**<mark style="background-color:red;">**impl塊要麼與trait的聲明在同一個的crate中，要麼與類型的聲明在同一個crate中**</mark><mark style="background-color:red;">。</mark>

也就是說，如果trait來自於外部crate，而且類型也來自於外部crate，編譯器不允許你為這個類型impl這個trait。**它們之中必須至少有一個是在當前crate中定義的**。

<mark style="color:red;">孤立規則是用來預防相依性專案，在新增新Trait實作時造成破壞</mark>，也就是說只有在當前區域Crate中而非外部Crate時，Rust才會允許Trait或是型別實作。

具體地說：

1. 如果要實現外部定義的 trait 需要先將其導入作用域。
2. 不允許對外部類型實現外部 trait；
3. 可以對外部類型實現自定義的 trait；
4. 可以對自定義類型上實現外部 trait。

外部是指不是由自己，而是由外部定義的，<mark style="color:red;">包括標准庫</mark>。

因為在其他的crate中，一個類型沒有實現一個trait，很可能是有意的設計。如果我們在使用其他的crate的時候，強行把它們“配對”，是會製造出bug的。

比如說，我們寫了一個程式，引用了外部庫lib1和lib2，lib1中聲明了一個trait T，lib2中聲明了一個struct S，我們不能在自己的程式中針對S實現T。這也意味著，上游開發者在給別人寫庫的時候，尤其要注意，一些比較常見的標準庫中的trait，如Display Debug ToString Default等，應該盡可能地提供好。否則，使用這個庫的下游開發者是沒辦法幫我們把這些trait實現的。

同理，如果是匿名impl，那麼這個impl塊必須與類型本身存在於同一個crate中。

### 可以對自定義類型上實現外部 trait與不可對外部類型實作外部trait

```rust
// 可以對自定義類型上實現外部 trait
// 外部的trait
use std::ops::Add;

// 本地的struct
struct Mydata(i32);

// 本地的struct實現外部的trait
impl Add<i32> for Mydata{
    type Output = i32;
    fn add(self, other:i32) -> Self::Output {
        (self.0) + other
    }
}

// error 不可為外部的類型實現trait
i// mpl Add<i32> for Option<Mydata>{
//    ...
//}

fn main(){
    let res = Mydata(3) + 5;
    println!("{res}");  // 8
}
```

### 1.41.0鬆綁Trait實作限制

但是這限制遇到泛型時，情況就變得複雜，例如當Crate定義了BetterVec結構，而開發者想要把該結構轉換成標準函式庫的Vec，程式碼寫作impl From\<BetterVec> for Vec{// ...}，在Rust 1.40中，這個寫法違反孤立原則，因為From和Vec都是定義在標準函式庫，對於當前Crate來說是外部的Trait與型別。

上述案例在Rust 1.41中，From和Vec仍然為外部的Trait與型別，但是Trait將會透過區域型別引數化，因此在最新版本這個實作是可行的。

## OOP語言的Interface與trait的區別

許多初學者會用自帶GC的語言中的“Interface”、抽象基類來理解trait這個概念，但是實際上它們有很大的不同。

Rust是一種使用者可以對記憶體有精確控制能力的強類型語言。我們可以自由指定一個變數是在堆疊裡面，還是在堆裡面，變數和指標也是不同的類型。

類型是有大小（實現了Sized trait）的。有些類型的大小是在編譯階段可以確定的，有些類型的大小是編譯階段無法確定的。<mark style="color:red;">目前版本的Rust規定，在函數參數傳遞、返回值傳遞等地方，都要求這個類型在編譯階段有確定的大小</mark>。否則，編譯器就不知道該如何生成程式碼了。

而trait本身既不是具體類型，也不是指標類型，它只是定義了針對類型的、抽象的“約束”。不同的類型可以實現同一個trait，滿足同一個trait的類型可能具有不同的大小。因此，<mark style="color:red;">trait在編譯階段沒有固定大小</mark>，目前我們不能直接使用trait作為執行個體變數、參數、返回值。

## 完整函式呼叫語法(Fully Qualified Syntax)

Fully Qualified Syntax提供一種無歧義的函式呼叫語法，允許程式師精確地指定想調用的是那個函數。以前也叫UFCS（universal function call syntax），也就是所謂的“通用函式呼叫語法”。

這個語法可以允許使用類似的寫法精確調用任何方法，包括成員方法和靜態方法。其他一切函式呼叫語法都是它的某種簡略形式。它的具體寫法為`<T asTraitName>::item`。

```rust
trait Cook {
    fn start(&self);
}
trait Wash {
    fn start(&self);
}
struct Chef;
impl Cook for Chef {
    fn start(&self) {
        println!("Cook::start");
    }
}
impl Wash for Chef {
    fn start(&self) {
        println!("Wash::start");
    }
}
fn main() {
    let me = Chef;
    //  無法分別是Cook或Wash traits的start
    // me.start();
    // 函數名字使用更完整的path來指定,
    // 同時,self參數需要顯式傳遞
    <dyn Cook>::start(&me);     // Cook::start
    <Chef as Cook>::start(&me); // Cook::start
    <Chef as Wash>::start(&me); // Wash::start
}
```

由此我們也可以看到，**所謂的“成員方法”也沒什麼特殊之處，它跟普通的靜態方法的唯一區別是，第一個參數是self，而這個self只是一個普通的函數參數而已**。

成員方法可以通過變數加小數點的方式調用。變數加小數點的調用方式在大部分情況下看起來更簡單更美觀，完全可以視為一種語法糖。

需要注意的是，通過小數點語法調用方法調用，有一個“隱藏著”的“取引用”步驟。雖然我們看起來原始程式碼長的是這個樣子me.start()，但是大家心裡要清楚，真正傳遞給start()方法的參數是\&me而不是me，這一步是編譯器自動幫我們做的。不論這個方法接受的self參數究竟是Self、\&Self還是\&mut Self，最終在源碼上，我們都是統一的寫法：variable.method()。而如果用UFCS語法來調用這個方法，我們就不能讓編譯器幫我們自動取引用了，必須手動寫清楚。

成員方法和普通函數其實沒什麼本質區別，只是多了一層名稱空間而已。

```rust
struct T(usize);
// membership function
impl T {
    fn get1(&self) -> usize {
        self.0
    }
    fn get2(&self) -> usize {
        self.0
    }
}
// function
fn get3(t: &T) -> usize {
    t.0
}
fn check_type(_: fn(&T) -> usize) {}
fn main() {
    // get1、get2和get3都可以自動轉成fn（&T）→usize類型
    check_type(T::get1);
    check_type(T::get2);
    check_type(get3);
}
```

## trait約束

Rust的trait的另外一個大用處是，作為**泛型約束**使用，<mark style="background-color:red;">**即限定只有實現給定trait的類型才可被調用**</mark>。

多個traits限制時，用`+`符號串接，有兩種語法：

* 在泛型的角括號中加入限制。
* 在函數後面加where(多trait限制時 較容易閱讀)。

```rust
use std::fmt::Debug;

// 限定實現Debug的類別T
fn my_print<T: Debug + Clone>(x: T) {
    println!("The value is {:?}.", x);
}
// 等價寫法，用where將限制放到後面
fn my_print<T>(x: T)
where
    T: Debug + Clone,
{
    println!("The value is {:?}.", x);
}
fn main() {
    my_print("Hello");
    my_print(41_i32);
    my_print(true);
    my_print(['a', 'b', 'c'])
}
```

## trait繼承

```rust
// trait允許繼承
trait Base {}
trait Derived: Base {}
// 等同於 trait Derived where Self: Base{}
struct T;
// 由於Derived繼承Base，因此impl時，
// 必須同時實現Base與Derived
impl Base for T {}
impl Derived for T {}
fn main() {}
```

這表示Derived trait繼承了Base trait。<mark style="color:red;">它表達的意思是，滿足Derived的類型，必然也滿足Base trait</mark>。所以，我們在針對一個具體類型implDerived的時候，編譯器也會要求我們同時impl Base。

在標準庫中，很多trait之間都有繼承關係。

```rust
trait Eq: PartialEq<Self> {}
trait Copy: Clone {}
trait Ord: Eq + PartialOrd<Self> {}
trait FnMut<Args>: FnOnce<Args> {}
trait Fn<Args>: FnMut<Args> {}
```

## Derive指令

Rust裡面為類型impl某些trait的時候，邏輯是非常機械化的。為許多類型重複而單調地impl某些trait，是非常枯燥的事情。為此，Rust提供了一個特殊的attribute，它可以幫我們自動impl某些trait。

```rust
/* 在你希望impl trait的類型前面寫#[derive（…）]，
 * 括弧裡面是你希望impl的trait的名字。這樣寫了之後，
 * 編譯器就幫你自動加上了impl塊
*/
#[derive(Copy, Clone, Default, Debug, Hash, PartialEq, Eq, PartialOrd, Ord)]
struct Foo {
    data: i32,
}
fn main() {
    let v1 = Foo { data: 0 };
    let v2 = v1;
    println!("{:?}", v2);
}
/* derive指令自動完成以下的內容
impl Copy for Foo { ... }
impl Clone for Foo { ... }
impl Default for Foo { ... }
impl Debug for Foo { ... }
impl Hash for Foo { ... }
impl PartialEq for Foo { ... }
......
*/
```

這些trait都是標準庫內部的較特殊的trait，它們可能包含有成員方法，但是成員方法的邏輯有一個簡單而一致的“範本”可以使用，編譯器就機械化地重複這個範本，幫我們實現這個預設邏輯。當然我們也可以手動實現。

目前，Rust支援的可以自動derive的trait有以下這些：`Debug`、`Clone`、`Copy`、`Hash`、`RustcEncodable`、`RustcDecodable`、`PartialEq`、`Eq`、`ParialOrd`、`Ord`、`Default`、`FromPrimitive`、`Send`、`Sync`。

## trait別名

跟type alias類似的，trait也可以起別名（trait alias）。假如在某些場景下，我們有一個比較複雜的trait：

```rust
pub trait Service {
    type Request;
    type Response;
    type Error;
    type Future: Future<Item = Self::Response, Error = Self::Error>;
    fn call(&self, req: Self::Request) -> Self::Future;
}
```

每次使用這個trait的時候都需要攜帶一堆的關聯類型參數。為了避、免這樣的麻煩，在已經確定了關聯類型的場景下，我們可以為它取一個別名：

```rust
trait HttpService =
    Service<Request = http::Request, 
            Response = http::Response, 
            Error = http::Error>;
```

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

##
