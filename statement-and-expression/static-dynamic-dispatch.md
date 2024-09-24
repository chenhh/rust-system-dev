# 靜態與動態分派

## 簡介

閉包或函數做為函數的參數時，可使用[靜態與動態分派](closure.md#jing-tai-yu-dong-tai-fen-pai)。

* 所謂“靜態分派”，是指具體調用哪個函數，在編譯階段就確定下來了。**Rust中的“靜態分派”靠泛型以及impl trait來完成**。對於不同的泛型類型參數，編譯器會生成不同版本的函數，在編譯階段就確定好了應該調用哪個函數。
* 所謂“動態分派”，是指具體調用哪個函數，在執行階段才能確定。**Rust中的“動態分派”靠特徵物件(Trait Object)來完成**。特徵物件本質上是指標，它可以指向不同的類型；指向的具體類型不同，調用的方法也就不同。

```rust
trait Bird {
    fn fly(&self);
}
struct Duck;
struct Swan;
// 兩個類別都實現Bird trait
impl Bird for Duck {
    fn fly(&self) {
        println!("duck duck");
    }
}
impl Bird for Swan {
    fn fly(&self) {
        println!("swan swan");
    }
}
// 靜態分派，使用泛型與限制
fn test_static<T: Bird>(arg: T) {
    arg.fly();
}
// 動態分派, 用Box將物件裝箱, 因為物件長度不確定，必須加上dyn關鍵字
fn test_dynamic(arg: Box<dyn Bird>) {
    arg.fly();
}

fn main() {
    let d = Duck;
    let s = Swan;
    test_static(d);
    test_static(s);

    let db = Box::new(Duck);
    let ds = Box::new(Swan);
    test_dynamic(db);
    test_dynamic(ds);
}
```

上例中，test\_dynamic函數的參數既可以是`Box<Duck>`類型，也可以是`Box<Swan>`類型，一樣實現了“多態”。但在參數類型這裡已經將“具體類型”資訊抹掉了，我們只知道它可以調用Bird trait的方法。而具體調用的是哪個版本的方法，實際上是由這個指標的值來決定的。這就是“動態分派”。

## 特徵物件(trait object)

指向trait的指標就是trait object。假如Bird是一個trait的名稱，那麼dyn Bird就是一個DST動態大小類型。

\&dyn Bird、\&mut dyn Bird、Box\<dyn Bird>、\*const dyn Bird、\*mut dyn Bird以及Rc\<dyn Bird>等等都是Trait Object。

trait object這個名字以後也會被改為dynamic trait type。impl Trait forTrait這樣的語法同樣會被改為impl Trait for dyn Trait。這樣能更好地跟impl Trait語法對應起來。

當指標指向trait的時候，這個指標就不是一個普通的指標了，變成一個“胖指標”。前文所講解的DST類型：

* 陣列類型\[T]是一個DST類型，因為它的大小在編譯階段是不確定的。
* 相對應的，&\[T]類型就是一個“胖指標”，它不僅包含了指標指向陣列的其中一個元素，同時包含一個長度資訊。它的內部表示實質上是Slice類型。

同理，Bird只是一個trait的名字，符合這個trait的具體類型可能有多種，這些類型並不具備同樣的大小，因此使用dyn Bird來表示滿足Bird約束的DST類型。指向DST的指標理所當然也應該是一個“胖指標”，它的名字就叫trait object。

比如`Box<dyn Bird>`，它的內部表示可以理解成下面這樣：

```rust
pub struct TraitObject {
    pub data: *mut (),
    pub vtable: *mut (),
}
```

它裡面包含了兩個成員，都是指向單元類型的裸指標。在這裡聲明的指標指向的類型並不重要，我們只需知道它裡面包含了兩個裸指標即可。由上可見，和Slice一樣，Trait Object除了包含一個指標之外，還帶有另外一個“中繼資料”，它就是指向“虛函數表”的指標。這裡用的是裸指標，指向unit類型的指標\*mut（）實際上類似於C語言中的void\*。

Rust的動態分派和C++的動態分派，記憶體佈局有所不同。在C++裡，如果一個類型裡面有虛函數，那麼每一個這種類型的變數內部都包含一個指向虛函數表的位址。而在Rust裡面，物件本身不包含指向虛函數表的指標，這個指標是存在於traitobject指標裡面的。如果一個類型實現了多個trait，那麼不同的traitobject指向的虛函數表也不一樣。

## object safe

trait object的構造是受到許多約束的，當這些約束條件不能滿足的時候，會產生編譯錯誤。

在以下條件下trait object是無法構造出來的：

1. 當trait有Self：Sized約束時
2. 當函數中有Self類型作為參數或者返回類型時
3. 當函數第一個參數不是self時
4. 當函數有泛型參數時

### 當trait有Self：Sized約束時

一般情況下，我們把trait當作類型來看的時候，它是不滿足Sized條件的。**因為trait只是描述了公共的行為，並不描述具體的內部實現，實現這個trait的具體類型是可以各種各樣的，佔據的空間大小也不是統一的**。Self關鍵字代表的類型是實現該trait的具體類型，在impl的時候，針對不同的類型，有不同的具體化實現。

```rust
trait Foo
where
    Self: Sized,
{
    fn foo(&self);
}
impl Foo for i32 {
    fn foo(&self) {
        println!("{}", self);
    }
}
fn main() {
    let x = 1_i32;
    x.foo();
    // 直接調用函數foo依然是可行的。
    // 可是，當我們試圖創建trait object的時候編譯器阻止了我們
    //let p = &x as &dyn Foo;
    //p.foo();
}
```

所以，如果我們不希望一個trait通過trait object的方式使用，可以為它加上Self：Sized約束。

同理，如果我們想阻止一個函數在虛函數表中出現，可以專門為該函數加上Self：Sized約束。

```rust
trait Foo {
    fn foo1(&self);
    fn foo2(&self)
    where
        Self: Sized;
}
impl Foo for i32 {
    fn foo1(&self) {
        println!("foo1 {}", self);
    }
    fn foo2(&self) {
        println!("foo2 {}", self);
    }
}
fn main() {
    let x = 1_i32;
    x.foo2();
    let p = &x as &dyn Foo;
    p.foo1();
    // p.foo2(); //有Sized限制，不可實現
}
```

### 當函數中有Self類型作為參數或者返回類型時

Self類型是個很特殊的類型，它代表的是impl這個trait的當前類型。編譯器不知道，因為它在編譯階段無法確定self指向的具體物件，它的類型是什麼只能在執行階段確定，無法在編譯階段確定。

Rust規定，如果函數中除了self這個參數之外，還在其他參數或者返回值中用到了Self類型，那麼這個函數就不是object safe的。這樣的函數是不能使用trait object來調用的。這樣的方法是不能在虛函數表中存在的。

這樣的規定在某些情況下會給我們造成一定的困擾。假如我們有下面這樣一個trait，它裡面的一部分方法是滿足object safe的，而另外一部分是不滿足的。如例子中因為new（）這個方法是不滿足object safe條件的。但是我們其實只想在trait object中調用double方法，並不指望通過traitobject調用new（）方法，但可惜編譯器還是直接禁止了這個trait object的創建。

我們可以通過加上Self: Sized的限制，把new（）方法從trait object的虛函數表中移除。

```rust
trait Double {
    fn new() -> Self where Self: Sized;
    fn double(&mut self);
}
impl Double for i32 {
    fn new() -> i32 {
        0
    }
    fn double(&mut self) {
        *self *= 2;
    }
}
fn main() {
    let mut i = 1;
    let p: &mut dyn Double = &mut i as &mut dyn Double;
    p.double();
}
```

### 當函數第一個參數不是self時

意思是，如果有“靜態方法”，那這個“靜態方法”是不滿足object safe條件的。這個條件幾乎是顯然的，編譯器沒有辦法把靜態方法加入到虛函數表中。

與上面講解的情況類似，如果一個trait中存在靜態方法，而又希望通過trait object來調用其他的方法，那麼我們需要在這個靜態方法後面加上Self：Sized約束，將它從虛函數表中剔除。

### 當函數有泛型參數時

```rust
trait SomeTrait {
    fn generic_fn<A>(&self, value: A);
}
```

trait object通過它調用成員的方法是通過vtable虛函數表來進行查找並調用。現在需要被查找的函數成了泛型函數，而泛型函數在Rust中是編譯階段自動展開的，generic\_fn函數實際上有許多不同的版本。這裡有一個根本性的衝突問題。Rust選擇的解決方案是，禁止使用trait object來調用泛型函數，泛型函數是從虛函數表中剔除了的。這個行為跟C++是一樣的。C++中同樣規定了類的虛成員函數不可以是template方法。

