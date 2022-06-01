# 泛型\(generics\)

## 簡介

泛型（Generics）是指把類型抽象成一種“參數”，資料和演算法都針對這種抽象的類型參數來實現，而不針對具體類型。當我們需要真正使用的時候，再具體化、產生實體類型參數。

## 資料結構中的泛型

```rust
enum Option<T> {
    Some(T),    // tuple struct
    None,
}

let x: Option<i32> = Some(42);
let y: Option<f64> = None;
```

這裡的&lt;T&gt;實際上是聲明了一個“類型”參數。在這個Option內部，Some（T）是一個tuple struct，包含一個元素類型為T。這個泛型參數類型T，可以在使用時指定具體類型。

泛型參數可以有多個也可以有預設值。

```rust
struct S<T = i32> {
    data: T,
}
fn main() {
    // 使用預設的type
    let v1 = S { data: 0 };
    // 指定T實現的type
    let v2 = S::<bool> { data: false };
    // 0 false
    println!("{} {}", v1.data, v2.data);
}
```

使用不同類型參數將泛型類型具體化後，獲得的是完全不同的具體類型。如Option&lt;i32&gt;和Option&lt;i64&gt;是完全不同的類型，不可通用，也不可相互轉換。當編譯器生成程式碼的時候，它會為每一個不同的泛型參數生成不同的程式碼。各種自訂複合類型都可以攜帶任意的泛型參數。

Rust規定，所有的泛型參數必須是真的被使用過的，否則會產生編譯錯誤。

```rust
// compile error, tpye T沒有被使用過
struct Num<T> {
    data: i32,
}
```

## 函數中的泛型

```rust
fn compare_option<T>(first: Option<T>, second: Option<T>) -> bool {
    match (first, second) {
        (Some(..), Some(..)) => true,
        (None, None) => true,
        _ => false,
    }
}
```

函數compare\_option有一個泛型參數T，兩個形參類型均為Option&lt;T&gt;。這意味著這兩個參數必須是完全一致的類型。如果我們在參數中傳入兩個不同的Option，會導致編譯錯誤。

```rust
fn main() {
    // 類型不匹配編譯錯誤
    println!("{}", compare_option(Some(1i32), Some(1.0f32)));
}
```

編譯器在看到這個函式呼叫的時候，會進行類型檢查：

* first的形參類型是Option&lt;T&gt;、實參類型是Option&lt;i32&gt;，
* second的形參類型是Option&lt;T&gt;、實參類型是Option&lt;f32&gt;。

這時編譯器的類型推導功能會進行一個類似解方程組的操作：由Option&lt;T&gt;==Option&lt;i32&gt;可得T==i32，而由Option&lt;T&gt;==Option&lt;f32&gt;又可得T==f32。這兩個結論產生了矛盾，因此該方程組無解，出現編譯錯誤。

如果我們希望參數可以接受兩個不同的類型，那麼需要使用兩個泛型參數：

```rust
fn compare_option<T1, T2>(
    first: Option<T1>, 
    second: Option<T2>) -> bool { ... }
```

一般情況下，調用泛型函數可以不指定泛型參數類型，編譯器可以通過類型推導自動判斷。某些時候，如果確實需要手動指定泛型參數類型，則需要使用function\_name：：&lt;type params&gt;（function params）的語法：

```rust
compare_option::<i32, f32>(Some(1), Some(1.0));
```

**Rust沒有C++那種無限制的ad hoc式的函數重載功能**。現在沒有，將來也不會有。主要原因是，這種隨意的函數重載對於程式碼的維護和可讀性是一種傷害。

通過泛型來實現類似的功能是更好的選擇。如果說，不同的參數類型，沒有辦法用trait統一起來，利用一個函數體來統一實現功能，那麼它們就沒必要共用同一個函數名。它們的區別已經足夠大，所以理應使用不同的名字。強行使用同一個函數名來表示區別非常大的不同函數邏輯，並不是好的設計。

我們還有另外一種方案，可以把不同的類型統一起來，那就是enum。**通過enum的不同成員來攜帶不同的類型資訊，也可以做到類似“函數重載”的功能。但這種做法跟“函數重載”有本質區別，因為它是有執行時開銷的。**

enum會在執行階段判斷當前成員是哪個變體，而“函數重載”以及泛型函數都是在編譯階段靜態分派的。同樣，Rust也不鼓勵大家僅僅為了省去命名的麻煩，而強行把不同類型用enum統一起來用一個函數來實現。如果一定要這麼做，那麼最好是有一個好的理由，而不僅是因為懶得給函數命名而已。

## impl區塊的泛型

impl的時候也可以使用泛型。在impl&lt;Trait&gt;for&lt;Type&gt;{}這個語法結構中，泛型類型既可以出現在&lt;Trait&gt;位置，也可以出現在&lt;Type&gt;位置。

與其他地方的泛型一樣，impl塊中的泛型也是先聲明再使用。在impl塊中出現的泛型參數，需要在impl關鍵字後面用尖括弧聲明。

當我們希望為某一組類型統一impl某個trait的時候，泛型就非常有用了。有了這個功能，很多時候就沒必要單獨為每個類型去重複impl了。

```rust
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

標準庫中的Into和From是一對功能互逆的trait。如果A：Into&lt;B&gt;，意味著B：From&lt;A&gt;。因此，標準庫中寫了這樣一段程式碼，意思是針對所有類型T，只要滿足U：From&lt;T&gt;，那麼就針對此類型impl Into&lt;U&gt;。有了這樣一個impl塊之後，我們如果想為自己的兩個類型提供互相轉換的功能，那麼只需impl From這一個trait就可以了，因為反過來的Intotrait標準庫已經幫忙實現好了。

## 泛型參數約束

Rust的泛型和C++的template非常相似，但也有很大不同。它們的最大區別在於執行類型檢查的時機。在C++裡面，範本的類型檢查是延遲到產生實體的時候做的。而在Rust裡面，泛型的類型檢查是當場完成的。

Rust採取了不同的策略，它會在分析泛型函數的時候當場檢查類型的合法性。這個檢查是怎樣做的呢？它要求用戶提供合理的“泛型約束”。在Rust中，trait可以用於作為“泛型約束”。

```rust
/* 由於泛型參數T沒有任何約束，
   因此編譯器認為a<b這個運算式是不合理的，
   因為它只能作用於支持比較運算子的類型。
   在Rust中，只有impl了PartialOrd的類型，才能支持比較運算子。
*/
fn max<T>(a: T, b: T) -> T {
    if a < b {
        b
    } else {
        a
    }
}
fn main() {
    let m = max(1, 2);
}
```

修復方案為泛型類型T添加泛型約束。泛型參數約束有兩種語法：

1. 在泛型參數聲明的時候使用冒號：指定；
2. 使用where子句指定。



```rust
use std::cmp::PartialOrd;
// 第一種寫法：在泛型參數後面用冒號約束
fn max<T: PartialOrd>(a: T, b: T) -> T {
// 第二種寫法,在後面單獨用 where 子句指定
fn max<T>(a: T, b: T) -> T
where T: PartialOrd
```

這兩種寫法達到的目的是一樣的。但是，在某些情況下（比如存在下文要講解的關聯類型的時候），where子句比參數聲明中的冒號約束具有更強的表達能力，但它在泛型參數列表中是無法表達的。

```rust
trait Iterator {
    type Item; // Item 是一個關聯類型
               // 此處的where子句沒辦法在聲明泛型參數的時候寫出來
    fn max(self) -> Option<Self::Item>
    where
        Self: Sized,
        Self::Item: Ord,
        // 它要求Self類型滿足Sized約束，
        // 同時關聯類型Self：：Item要滿足Ord約束，
        // 這是用冒號語法寫不出來的。
    {
    }
}
```

在有了“泛型約束”之後，編譯器不僅會在聲明泛型的地方做類型檢查，還會在產生實體泛型的地方做類型檢查。

## 關聯類型

trait中不僅可以包含方法（包括靜態方法）、常量，還可以包含“類型”。

```rust
pub trait Iterator {
    type Item;
    // ...
}
```

這樣在trait中聲明的類型叫作“關聯類型”（associated type）。關聯類型也同樣是這個trait的“泛型參數”。只有指定了所有的泛型參數和關聯類型，這個trait才能真正地具體化。

```rust
use std::fmt::Debug;
use std::iter::Iterator;
// 。跟普通泛型參數比起來，關聯類型參數必須使用名字賦值的方式。
fn use_iter<ITEM, ITER>(mut iter: ITER)
where
    ITER: Iterator<Item = ITEM>,
    ITEM: Debug,
{
    while let Some(i) = iter.next() {
        println!("{:?}", i);
    }
}
fn main() {
    let v: Vec<i32> = vec![1, 2, 3, 4, 5];
    use_iter(v.iter());
}
```

假如我們想設計一個泛型的“圖”類型，它包含“頂點”和“邊”兩個泛型參數，如果我們把它們作為普通的泛型參數設計，那麼看起來就是：

```rust
trait Graph<N, E> {
    fn has_edge(&self, node1: &N, node2: &N) -> bool;
    // ...
}
```

現在如果有一個泛型函數，要計算一個圖中兩個頂點的距離，它的簽名會是：

```rust
fn distance<N, E, G: Graph<N, E>>(graph: &G, start: &N, end: &N) -> uint {
    //...
}
```

### 關聯類型優點：可簡化函數參數

我們可以看到，泛型參數比較多，也比較麻煩。對於指定的Graph類型，它的頂點和邊的類型應該是固定的。在函數簽名中再寫一遍其實沒什麼道理。如果我們把普通的泛型參數改為“關聯類型”設計，那麼資料結構就成了：

```rust
trait Graph<N, E> {
    type N;
    type E;
    fn has_edge(&self, node1: &N, node2: &N) -> bool;
    // ...
}
```

對應的，計算距離的函數簽名可以簡化成：

```rust
fn distance<G>(graph: &G, start: &G::N, end: &G::N) -> uint
where G: Graph { //...}
```

### trait的impl匹配規則

假如我們要設計一個trait，名字叫作ConvertTo，用於類型轉換。那麼，我們就有兩種選擇。一種是使用泛型類型參數，另一種使用關聯類型：

```rust
// 使用泛型
trait ConvertTo<T> {
    fn convert(&self) -> T;
}
// 使用關聯類型
trait ConvertTo {
    type DEST;
    fn convert(&self) -> Self::DEST;
}
```

如果我們想寫一個從i32類型到f32類型的轉換，在這兩種設計下，程式碼分別是：

```rust
// 使用泛型
impl ConvertTo<f32> for i32 {
    fn convert(&self) -> f32 { *self as f32 }
}
// 使用關聯類型
impl ConvertTo for i32 {
    type DEST = f32;
    fn convert(&self) -> f32 { *self as f32 }
}
```

目前為止，這兩種設計似乎沒什麼區別。但是，假如我們想繼續增加一種從i32類型到f64類型的轉換，使用泛型參數來實現的話，可以編譯通過，但關聯類型會出現編譯錯誤。

```rust
// 泛型可成功編譯
impl ConvertTo<f64> for i32 {
    fn convert(&self) -> f64 { *self as f64 }
}
// compile error, conflicting implementations of trait `ConvertTo` for type `i32`
impl ConvertTo for i32 {
    type DEST = f64;
    fn convert(&self) -> f64 { *self as f64 }
}
```

由此可見，**如果我們採用了“關聯類型”的設計方案，就不能針對這個類型實現多個impl**。

在編譯器的眼裡，如果trait有類型參數，那麼給定不同的類型參數，它們就已經是不同的trait，可以同時針對同一個類型實現impl。如果trait沒有類型參數，只有關聯類型，給關聯類型指定不同的類型參數是不能用它們針對同一個類型實現impl的。

## 何時使用關聯類型

雖然關聯類型也是類型參數的一種，但它與泛型類型參數列表是不同的。我們可以把這兩種泛型類型參數分為兩個類別：

* 輸入類型參數：在尖括弧中存在的泛型參數，是輸入類型參數；
* 輸出類型參數：在trait內部存在的關聯類型，是輸出類型參數。

輸入類型參數是用於決定匹配哪個impl版本的參數；輸出類型參數則是可以由輸入類型參數和Self類型決定的類型參數。

```rust
trait ConvertTo<T> {
    fn convert(&self) -> T;
}
/* 我們可以用不同的參數類型實現重載，
   但是不能用不同的返回類型來做重載，
   因為編譯器是根據參數類型來判斷調用哪個版本的重載函數的，
   而不是依靠返回值的類型。
*/

impl ConvertTo<f32> for i32 {
    fn convert(&self) -> f32 {
        *self as f32
    }
}
impl ConvertTo<f64> for i32 {
    fn convert(&self) -> f64 {
        *self as f64
    }
}
fn main() {
    let i = 1_i32;
    // compile error, 無法判定使用f32或f64的版本
    // let f = i.convert(); 
    // 要明確指定f的類型
    let f: f32 = i.convert();
    println!("{:?}", f);
}
```

## 泛型特化

Rust2018目前還不穩定，暫不介紹。



