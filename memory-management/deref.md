# 解引用(deref)

## 簡介

引用是對記憶體塊的借用，Rust裡每一個記憶體塊都是有主人的，主人就是對記憶體擁有所有權的變數，<mark style="color:blue;">**沒有主人(無變數綁定)的記憶體塊我們稱之為記憶體洩露(memory leak)**</mark>。

解引用(Deref)是引用(Ref)的反操作。比如說，我們有引用型別`let p: &T`;，那麼可以用`*`符號執行解引用操作，`let v: T = *p`_;。如果`p`的型別是`&T`， 那麼`*`_`p`的型別就是`T`。

Deref和DerefMut都是Rust中的特徵，用來對指標型別進行轉化，得到指標所指向的內容。比如從Box或Rc中得到T，或是從String中得到\&str<mark style="color:red;">。另外我們經常使用的解引用運算子</mark><mark style="color:red;">`*`</mark><mark style="color:red;">其實就是呼叫了deref函數</mark>。

<mark style="background-color:blue;">**解引用是通過引用找到記憶體塊真正的主人(引用存放的記憶體地位，和指標存放內容相同；解引用是將此記憶體地位之值取出)**</mark>，然後你可以跟主人借一些不同型別的引用，比如從`&mut T`借成`Pin<&mut T>`，此行為和C語言中的指標一致，因此可以將引用類型解釋為指標。

* 因為rust有自動deref機制，<mark style="color:red;">所以從&</mark><mark style="color:red;">`T`</mark><mark style="color:red;">到</mark><mark style="color:red;">`T`</mark><mark style="color:red;">的轉換一定只有deref這一種方式</mark>。
* 但是如果在一個需要`&T`的場合傳入一個`T`，邏輯上你有可能想要把這個`T`通過某種方式轉換到`&T`，而不是傳入對這個`T`的借用，所以這裡是不能自動的。
* 例如：如果函數需要`&st`r類型的參數，你給它一個`&&str`，編譯器會把自動把它解引用，變成`&str`。
* 反過來，如果函數要一個`&&str`，你給它一個`&str`，編譯器解一次引用，結果是`str`，這不能再解引用了，因此編譯失敗。

這個主人可能被包了很多層，當你以為你找了主人，其實它只是個皮，所以會存在不斷解引用的情況，你可能需要加很多個`*`符號，當然很多個`*`符號不符合設計美感。

`例如let t=&*v`說明`v`是可以解引用的。

* 這裡的解引用可能是的普通引用的解引用（這時可得`t=v`)；
* 也可能`v`是一個裸指標，`&*v`下來相當於把不安全的裸指標轉換為普通的引用；
* 也可能`v`是一個智慧指標，比如`Box<v>`，那`&*v`就相當於解出`&T`了；
* 還有可能是`v`的類型實現了Deref trait，那`&*v`_就相當於呼叫&_(v.deref())
* <mark style="color:red;">`*`</mark><mark style="color:red;">引用運算子對於沒有實現Dered的引用物件，返回其原始的物件</mark>。
* 引用運算子對於實現Dered Trait的（非引用物件）操作是隱式的呼叫 其`deref()`方法，也就是從一個引用類型到另外一個引用類型。

### 常規引用的解引用

```rust
fn main() {
    let x = 5;
    let y = &x; // 一般的不可變引用

    assert_eq!(5, x);
    assert_eq!(5, *y); // 必須手動解引用才能比較值
}
```

### 智慧指標的解引用

```rust
use std::ops::Deref;

fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);           // 自動解引用
    assert_eq!(5, *(y.deref())); // 手動解引用
}
```

```rust
// intrinsics只能在nightly頻道使用
#![feature(core_intrinsics)]
fn print_type_of<T>(_: &T) {
    println!("{}", unsafe { std::intrinsics::type_name::<T>() });
}

fn main() {
    let v1 = 1;
    let p = &v1; //取引用操作
    println!("{:p} {:p}", &v1, p); //兩者指向相同的記憶體地位
    let v2 = *p; //解引用操作
    println!("{v1} {v2}"); // 1, 1
    print_type_of(&v1); // i32
    print_type_of(&p); // &i32
    print_type_of(&v2); // i32
}
```

* Rust中允許一部分運算子可以由用戶自訂行為，即“運算子重載”。其中“解引用”是一個非常重要的操作符，它允許重載。
* 而需要注意的是，“取引用”操作符，如`&`、`&mut`，是不允許重載的。因此，“取引用”和“解引用”並非對稱互補關係。**`*&T`的類型一定是`T`，而`&*T`的類型未必就是`T`**。
* **更重要的是，**<mark style="background-color:red;">**在某些情況下，編譯器幫我們插入了自動解引用(deref)的呼叫，簡化程式碼**</mark>。
* 在Deref的基礎上，我們可以封裝出一種自訂類型，它可以直接調用其內部的其他類型的成員方法，我們可以把這種類型稱為智慧指標類型。

## 自動解引用和手動解引用

當我們對智慧指標 Box 進行解引用時，實際上 Rust 為我們呼叫了以下方法：

```rust
*(y.deref())
```

首先呼叫 deref 方法返回值的常規引用，然後通過`*`對常規引用進行解引用，最終獲取到目標值。

至於 Rust 為何要使用這個有點囉嗦的方式實現，<mark style="color:red;">原因在於所有權系統的存在</mark>。如果 deref 方法直接返回一個值，而不是引用，那麼該值的所有權將被轉移給呼叫者，而我們不希望呼叫者僅僅只是 \*T 一下，就拿走了智慧指針中包含的值。

需要注意的是，`*` 會無限遞迴替換，從 `*y` 到 `*(y.deref())` 只會發生一次，而不會繼續進行替換然後產生形如 `*((y.deref()).deref())` 的怪物。

* <mark style="color:red;">自動解引用</mark>：Rust為了減少某些場合下重複解引用導致的程式碼美觀問題，在編譯期做了一些智慧識別功能，比如帶有`&T`引數的函式被呼叫的時候，你傳`&&&......&&&T`都可以自動解引用，直到符合函式的引數型別為止(註：解引用簡單的說就是&&\&T->&\&T->\&T->T每一層都少一個&後，檢查看看有沒有符合程式需要的條件)。
* <mark style="color:red;">手動解引用</mark>，就是和其他語言類似，<mark style="background-color:orange;">借用是</mark><mark style="background-color:orange;">`&`</mark><mark style="background-color:orange;">操作符，解引用是</mark><mark style="background-color:orange;">`*`</mark><mark style="background-color:orange;">操作符</mark>。

我們也可以通過自行實現`Deref`這個Trait來自定義解引用的最終目標是什麼，而恰恰這個也是Rust語言最難的地方，你得瞭解每個型別是否實現了Deref，而Rust型別實在是太多了，連\&T借用也算一個新的型別，\&T是不能繼承T的所有特性的。

```rust
fn hello(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let m = Box::new(String::from("Rust"));
    // 自動轉換由&Box(String) -> &String -> &str
    hello(&m);
}
```

```rust
fn print_arr(arr: &[i32]){
    println!("{:?}", arr);
} 

fn main() {
    let a = vec![1,2,3,4];
    print_arr(&&&&&&&a); // 自動解引用至&[i32]
}

fn print_arr(arr: &[&str]){
    println!("{:?}", arr);
} 

fn main() {
    let a = vec!["hello","world","to","you"];
    print_arr(&&&&a);
}
```

## 自訂解引用

解引用操作可以被自訂。方法是，實現標準庫中的`std::ops::Deref`或者`std::ops::DerefMut`這兩個trait。 `DerefMut`的唯一區別是返回的是`&mut`型引用都是類似的，因此不過多介紹了。

```rust
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

請大家注意這裡的型別，`deref()` 方法返回的型別是 `&Target`，而不是 `Target`。

這個trait有一個關聯類型Target，代表解引用之後的目標類型。 比如，<mark style="background-color:red;">標準庫中實現了</mark><mark style="background-color:red;">`String`</mark><mark style="background-color:red;">向</mark><mark style="background-color:red;">`str`</mark><mark style="background-color:red;">的解引用轉換</mark>：

```rust
impl ops::Deref for String {
    type Target = str;
    #[inline]
    fn deref(&self) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}
```

如果說有變數`s`的類型為`String`，`*s` 的類型並不等於 `s.deref()` 的類型。 `*s`的類型實際上是 `Target`，即`str`。`&*s`的型別為`&str`。 而 `s.deref()` 的型別為 `&Target`，即 `&str`。

## 標準庫的容器

標準庫中有許多我們常見的類型實現了這個Deref操作符。比如`Vec<T>`、`String`、`Box<T>`、`Rc<T>`、`Arc<T>`等。它們都支援“解引用”操作。

從某種意義上來說，它們都可以算做特種形式的“指標”（像胖指標一樣，是帶有額外中繼資料的指標，只是中繼資料不限制在`usize`範圍內了）。我們可以把這些類型都稱為“智慧指標”。比如我們可以這樣理解這幾個類型：

* `Box<T>`是“指標”，指向一個在堆積上分配的物件；
* `Vec<T>`是“指標”，指向一組同類型的順序排列的堆上分配的物件，且攜帶有當前緩存空間總大小和元素個數大小的中繼資料；
* `String`是“指標”，指向的是一個堆上分配的位元組陣列，其中保存的內容是合法的utf8字元序列。且攜帶有當前緩存空間總大小和字串實際長度的中繼資料。

<mark style="color:red;">以上幾個類型都對所指向的內容擁有</mark><mark style="color:red;">**所有權**</mark><mark style="color:red;">，管理著它們所指向的記憶體空間的分配和釋放</mark>。`·Rc<T>`和`Arc<T>`也是某種形式的、攜帶了額外中繼資料的“指標”，它們提供的是一種“共用”的所有權，當所有的引用計數指標都銷毀之後，它們所指向的記憶體空間才會被釋放。

自訂解引用操作符可以讓用戶自行定義各種各樣的“智慧指標”，完成各種各樣的任務。再配合上編譯器的“自動”解引用機制，非常有用。

## 自動解引用

自動解引用是為了方便程式設計師，在變數不滿足條件的情況下，自動對變數使用解引用。

比如rust的庫中針對\&str實現了很多字串的操作，然而並沒有針對string進行實現，但是我們可以直接使用string進行字串操作，實際上是編譯器自動把\&string解引用為了\&str。

### 三種 Deref 強制轉換

Rust 在發現類型和 trait 實現滿足三種情況時會進行 Deref 強制轉換：

* 當 `T: Deref<Target=U>` 時從 `&T` 到 `&U`。
* 當 `T: DerefMut<Target=U>` 時從 `&mut T` 到 `&mut U`。
* 當 `T: Deref<Target=U>` 時從 `&mut T` 到 `&U`。

頭兩個情況除了可變性之外是相同的：第一種情況表明如果有一個 `&T`，而 `T` 實現了返回 `U` 類型的 Deref，則可以直接得到 `&U`。第二種情況表明對於可變引用也有著相同的行為。

第三個情況有些微妙：Rust 也會將可變引用強轉為不可變引用。但是反之是 不可能 的：不可變引用永遠也不能強轉為可變引用。因為根據借用規則，如果有一個可變引用，其必須是這些數據的唯一引用（否則程式將無法編譯）。將一個可變引用轉換為不可變引用永遠也不會打破借用規則。將不可變引用轉換為可變引用則需要初始的不可變引用是數據唯一的不可變引用，而借用規則無法保證這一點。因此，Rust 無法假設將不可變引用轉換為可變引用是可能的。

```rust
fn main() {
    let s = "hello";
    println!("length: {}", s.len());    // str::len(&s)
    // 以下均為自動解引用
    println!("length: {}", (&s).len()); // 等價於 str::len(&&s)
    println!("length: {}", (&&&&&&&&&&&&&s).len());
}
```

len方法的簽名為：

```rust
fn len(&self) -> usize
```

它接受的receiver參數是`&str`，因此我們可以用UFCS語法調用：

```rust
println!("length: {}", str::len(&s));
```

但是，如果我們使用`&&&&&&&&&&str`類型來調用成員方法，也是可以的。**原因就是，Rust編譯器幫我們做了隱式的deref調用，當它找不到這個成員方法的時候，會自動嘗試使用deref方法後再找該方法，一直迴圈下去**。

編譯器在`&&&str`類型裡面找不到`len`方法；嘗試將它deref，變成`&&str`類型後再尋找len方法，還是沒找到；繼續deref，變成`&str`，現在找到len方法了，於是就調用這個方法。

**自動deref的規則是，如果類型T可以解引用為U，即T：Deref\<U>，則\&T可以轉為\&U**。

### 自動解引用的例子：Rc

```rust
use std::rc::Rc;
fn main() {
    // 我們創建了一個指向String類型的Rc指標，並調用了bytes（）方法
    // 但是Rc類別並沒有bytes()方法
    let s = Rc::new(String::from("hello"));
    println!("{:?}", s.bytes());
}

// 自動解引用展開如下
use std::ops::Deref;
use std::rc::Rc;
fn main() {
    let s = Rc::new(String::from("hello"));
    println!("length: {}", s.len());
    println!("length: {}", s.deref().len());
    println!("length: {}", s.deref().deref().len());
    println!("length: {}", (*s).len());
    println!("length: {}", (&*s).len());
    println!("length: {}", (&**s).len());
}
```

這裡的機制是這樣的：

* `Rc`類型本身並沒有`bytes()`方法，所以編譯器會嘗試自動deref，試試`s.deref().bytes()`。
* `String`類型其實也沒有`bytes()`方法，但是`String`可以繼續deref，於是再試試`s.deref().deref().bytes()`。
* 這次在`str`類型中找到了`bytes()`方法，於是編譯通過。

我們實際上通過`Rc`類型的變數調用了`str`類型的方法，讓這個智慧指標透明。這就是自動Deref的意義。

這就是為什麼`String`需要實現`Deref` trait，是為了讓`&String`類型的變數可以在必要的時候自動轉換為`&str`類型。所以`String`類型的變數可以直接調用`str`類型的方法。

```rust
/* 雖然s的類型是String，
   但它在調用bytes()方法的時候，
   編譯器會自動查找並轉換為s.deref().bytes()調用。
   所以String類型的變數就可以直接調用str類型的方法了。
*/
let s = String::from("hello");
let len = s.bytes();
```

同理：`Vec<T>`類型也實現了Deref trait，目標類型是`[T]`，`&Vec<T>`類型的變數就可以在必要的時候自動轉換為`&[T]`陣列切片類型；`Rc<T>`類型也實現了`Deref` trait，目標類型是`T`，`Rc<T>`類型的變數就可以直接調用`T`類型的方法。

### &\*連寫法分開寫意義不同

因為編譯器很聰明，它看到`&*`這兩個操作連在一起的時候，會直接把`&*s`運算式理解為`s.deref()`，這時候`p`只是`s`的一個借用而已。而如果把這兩個操作分開寫，會先執行`*s`把內部的資料move出來，再對這個臨時變數取引用，這時候`s`已經被移走了，生命週期已經結束。

從這裡我們也可以看到，預設的“取引用”、“解引用”操作是互補抵消的關係，互為逆運算。但是，在Rust中，只允許自訂“解引用”，不允許自訂“取引用”。如果類型有自訂“解引用”，那麼對它執行“解引用”和“取引用”就不再是互補抵消的結果了。先`&`後`*`以及先`*`後`&`的結果是不同的。

## 參考資料

* [\[知乎\]rust解引用再引用是什麼操作？](https://www.zhihu.com/question/403915322)
* [rust course：Deref 解引用](https://course.rs/advance/smart-pointer/deref.html)
