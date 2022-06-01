# 閉包(closure)

## 簡介

closure看起來和普通函數很相似，但實際上它們有許多區別。

* <mark style="background-color:red;">**最主要的區別是，closure可以“捕獲”外部環境變數，而fn不可以，fn要使用外部變量只能由參數傳入，且要考慮是move, copy還是borrow語意**</mark><mark style="background-color:red;">。</mark>對於不需要捕獲環境變數的場景，普通函數fn和closure是可以互換使用的。
* fn定義和調用的位置並不重要，Rust中是不需要前向聲明的。只要函式定義在當前範圍內是可以觀察到的，就可以直接調用，不管在源碼內的相對位置如何**。相對而言，closure更像是一個變數，它具有和變數同樣的“生命週期”**。

閉包（closure）是一種匿名函數，具有“捕獲”外部變數的能力。閉包有時候也被稱作lambda運算式(C++)。它有兩個特點：

1. 可以像函數一樣被調用；
2. 可以捕獲當前環境中的變數。

```rust
fn main() {
    // 閉包有兩個參數，以兩個|包圍。執行語句包含在{}中。
    // 閉包的參數和返回數值型別的指定與普通函數的語法相同。
    let add = |a: i32, b: i32| -> i32 {
        a + b
    };
    let x = add(1, 2);
    println!("result is {x}");
    
    // 閉包的參數和返回數值型別都是可以省略的
    let add2 = |a, b| { a + b };
    let x2 = add2(2,3);
    println!("result2 is {x2}");
    
    // 如果閉包的語句體只包含一條語句，
    // 那麼外層的大括弧也可以省略；如果有多條語句則不能省略。
    let add3 = |a, b|  a + b ;
    let x3 = add3(3,4);
    println!("result3 is {x3}");
}
```

```rust
fn main() {
    let x = 1_i32;
    // closure可以訪問同區間的變數，
    // 而一般的fn做不到
    let inner_add = || x + 1;
    let x2 = inner_add();
    println!("result is {x2}");
}
```

## 變數捕獲

Rust目前的closure實現，又叫作unboxed closure，它的原理與C++11的lambda非常相似。<mark style="background-color:green;">當一個closure創建的時候，編譯器幫我們生成了一個匿名struct類型，通過自動分析closure的內部邏輯，來決定該結構體包括哪些資料，以及這些資料該如何初始化</mark>。

```rust
fn main() {
    let x = 1_i32;
    let add_x = |a| x + a;
    let result = add_x(5);
    println!("result is {result}");
}
```

以struct模擬如下。實際上Rust編譯器就是用類似的手法來處理閉包語法的。對比一下使用閉包語法的版本和手動實現的版本，我們可以看到，創建閉包的時候，就相當於創建了一個結構體，我們把需要捕獲的環境變數存到這個結構體中。**閉包調用的時候，相當於調用了跟這個結構體相關的一個成員函數**。

```rust
struct Closure {
    inner1: i32,
}
impl Closure {
    fn call(&self, a: i32) -> i32 {
        self.inner1 + a
    }
}
fn main() {
    let x = 1_i32;
    let add_x = Closure { inner1: x };
    let result = add_x.call(5);
    println!("result is {}", result);
}
```

但是，還有幾個問題沒有解決。當編譯器把閉包語法糖轉換為普通的類型和函式呼叫的時候：

1. 結構體內部的成員應該用什麼類型，如何初始化？應該用i32或是\&i32還是\&mut i32？
2. 函式呼叫的時候self應該用什麼類型？應該寫self或是\&self還是\&mut self？

Rust主要是通過分析外部變數在閉包中的使用方式，通過一系列的規則自動推導出來的。主要規則如下：

1. 如果一個外部變數在閉包中，只通過借用指標&使用，那麼這個變數就可通過引用&的方式捕獲；
2. 如果一個外部變數在閉包中，通過\&mut指標使用過，那麼這個變數就需要使用\&mut的方式捕獲；
3. 如果一個外部變數在閉包中，通過所有權轉移的方式使用過，那麼這個變數就需要使用“by value”self的方式捕獲。

簡單點總結規則是，在保證能編譯通過的情況下，編譯器會自動選擇一種對外部影響最小的類型存儲。<mark style="background-color:red;">**對於被捕獲的類型為T的外部變數，在匿名結構體中的存儲方式選擇為：盡可能先選擇\&T類型，其次選擇\&mut T類型，最後選擇T類型**</mark><mark style="background-color:red;">。</mark>

```rust
struct T(i32);
fn by_value(_: T) {println!("by vaule");}
fn by_mut(_: &mut T) {println!("by mut");}
fn by_ref(_: &T) {println!("by ref");}
fn main() {
    let x: T = T(1);
    let y: T = T(2);
    let mut z: T = T(3);
    let closure = || {
        by_value(x);
        by_ref(&y);
        by_mut(&mut z);
    };
    closure();
}
```

## move關鍵字

以上變數捕獲的規則都是針對只作為區域變數的閉包而準備的。

**有些時候，我們的閉包的生命週期可能會超過一個函數的範圍。**比如，我們可以將此閉包存儲到某個資料結構中，在當前函數返回之後繼續使用。

這樣一來，就可能出現更複雜的情況：在閉包被創建的時候，它通過引用的方式捕獲了某些區域變數，而在閉包被調用的時候，它所指向的一些外部變數已經被釋放了。

<mark style="background-color:red;">**closure加上move關鍵字之後，所有的變數捕獲全部使用by value的方式**</mark><mark style="background-color:red;">。</mark>

```rust
/*
  函數make_adder中有一個區域變數x，
  按照前面所述的規則，它被閉包所捕獲，
  而且可以使用引用&的方式完成閉包內部的邏輯，
  因此它是被引用捕獲的。而閉包則作為函數返回值被傳遞出去了。
  於是，閉包被調用的時候，它內部的引用所指向的內容已經被釋放了。
  這種情況，應該會出現典型的野指標問題，屬於記憶體不安全的範疇。
*/
fn make_adder(x: i32) -> Box<Fn(i32) -> i32> {
    // 加上move關鍵字之後，所有的變數捕獲全部使用by value的方式
    Box::new(move |y| x + y)
}
fn main() {
    let f = make_adder(3);
    println!("{}", f(1)); // 4
    println!("{}", f(10)); // 13
}
```

## 閉包作為函數輸入參數

果我們想讓閉包作為一個參數傳遞到函數中，可以這樣寫：

```rust
fn call_with_closure<F>(some_closure: F) -> i32
where
    F: Fn(i32) -> i32,
{
    some_closure(1)
}
fn main() {
    let answer = call_with_closure(|x| x + 2);
    println!("{}", answer); // 3
}
```

其中泛型參數F的約束條件是`F：Fn（i32）->i32`。<mark style="background-color:red;">這裡</mark><mark style="background-color:red;">`Fn（i32）->i32`</mark><mark style="background-color:red;">是針對閉包設計的專門的語法</mark>。這樣設計為了讓它們看起來跟普通函數類型`fn（i32）->i32`更相似。

當以閉包作為輸入參數時，必須指出閉包的完整類型，它是通過使用以下 trait 中的一種來指定的。其受限制程度按以下順序遞減：事<mark style="background-color:red;">實上是在滿足使用需求的前提下盡量以限制最多的方式捕獲</mark>。

* `Fn`：表示捕獲方式為通過引用（`&T`）的閉包。
* `FnMut`：表示捕獲方式為通過可變引用（`&mut T`）的閉包。
* `FnOnce`：表示捕獲方式為通過值（`T`）的閉包。

一定要注意的是：每個閉包，編譯器都會為它生成一個匿名結構體類型；即使兩個閉包的參數和返回值一致，它們也是完全不同的兩個類型，只是都實現了同一個trait而已。

```rust
fn main() {
    // 同一個變數綁定了兩次, error
    let mut closure = |x: i32| -> i32 { x + 2 };
    closure = |x: i32| -> i32 { x - 2 };
    println!("{}", closure());
}
```

## 靜態與動態分派

與普通的trait一樣，如果我們需要向函數中傳遞閉包，有下面兩種方式。

* 通過泛型的方式。這種方式會為不同的閉包參數類型生成不同版本的函數，實現**靜態分派**。
* 通過trait object的方式。這種方式會將閉包裝箱進入堆積記憶體中，向函數傳遞一個胖指標，實現**運行期動態分派**。

<mark style="color:red;">如果我們希望一個閉包作為函數的返回值，那麼就不能使用泛型的方式了</mark>。因為如果泛型類型不在參數中出現，而僅在返回類型中出現的話，會要求在調用的時候顯式指定類型，編譯器才能完成類型推導。可是調用方根本無法指定具體類型，因為閉包類型是匿名類型，用戶無法顯式指定。

```rust
// 這裡是泛型參數。對於每個不同類型的參數,編譯器將會生成不同版本的函數
fn static_dispatch<F>(closure: &F)
where
    F: Fn(i32) -> i32,
{
    println!("static dispatch {}", closure(42));
}
// 這裡是 `trait object``Box<Fn(i32)->i32>`也算`trait object`。
fn dynamic_dispatch(closure: &Fn(i32) -> i32) {
    println!("dynamic dispatch {}", closure(42));
}
fn main() {
    let closure1 = |x| x * 2;
    let closure2 = |x| x * 3;
    fn function_ptr(x: i32) -> i32 {
        x * 4
    };
    static_dispatch(&closure1);
    static_dispatch(&closure2);
    static_dispatch(&function_ptr);
    // 普通`fn`函數也實現了`Fn trait`,它可以與此參數類型匹配。`fn`不可以捕獲外部變數
    dynamic_dispatch(&closure1);
    dynamic_dispatch(&closure2);
    dynamic_dispatch(&function_ptr);
}
```

如果我們希望一個閉包作為函數的返回值，那麼就不能使用泛型的方式了。因為如果泛型類型不在參數中出現，而僅在返回類型中出現的話，會要求在調用的時候顯式指定類型，編譯器才能完成類型推導。

```rust
fn test() -> Box<dyn Fn(i32) -> i32> {
    let c = |i: i32| i * 2;
    Box::new(c)
}
fn main() {
    let closure = test();
    let r = closure(2);
    println!("{}", r);  // 4
}
```

## 閉包與生命週期

\[todo]
