# 智慧指標(smart pointer)

## 簡介

Rust 中最常見的指標是引用（reference）。引用以`&`符號為標志並借用了他們所指向的值。除了引用數據沒有任何其他特殊功能。它們也沒有任何額外開銷，所以應用得最多。

另一方面，智慧指標（smart pointers）是一類資料結構，他們的表現類似指標，但是也擁有額外的元數據和功能(如len()可知長度)。

在 Rust 中，普通引用和智慧指針的一個額外的區別是<mark style="background-color:red;">引用是一類只借用(borrow)資料的指標；相反，在大部分情況下，智慧指標「擁有」(有所有權) 他們指向的資料。</mark>

智慧指<mark style="background-color:red;">標</mark>通常使用結構體(struct)實現。智慧指標區別於常規結構體的顯著特性在於其實現了 `Deref` 和 `Drop` trait。

* `Deref` trait 允許智慧指標結構體實例表現的像引用一樣，這樣就可以編寫既用於引用、又用於智慧指針的程式碼。
* `Drop` trait 允許我們自定義當智慧指標離開作用域時運行的程式碼。

## 如何使用 struct 模擬指標

必須做到兩件事：

* 確認解引用(dereference)時可解開引用，取得底層值 ，即實作 `Deref` trait。&#x20;
* 確認指標結束生命週期時，會正確釋放資源，即實作`Drop` trait。

符合上述要件，並在 safe Rust 前提下實作的智慧指標，就可消弭絕大多數記憶體問題，例如 double free 或 wild pointer。

<mark style="background-color:red;">Smart Pointer in Rust ＝ Data structure + Deref + Drop</mark>

### Deref trait

* 用來多載解引用運運算元 \*（dereference）的 trait。&#x20;
* 有一個 required method `fn deref(&self) -> &Self::Target`&#x20;
* DerefMut 則是用在 `&mut` 的 dereference。&#x20;
* Rust 有很多 implicit dereference，實作 Deref trait 會簡單很多。

### 自動解引用(Implicit dereference)

為了方便寫程式，不用再區分變數(obj.method)、變數指標(obj->method)、指標解引用((\*obj).method)，Rust 幫我們統一介面，只要是函式傳參或方法呼叫，當：

* 對指向 value type 的指標操作時，就執行原本的動作。&#x20;
* 超過一層指標包裹 value 時，會先呼叫 obj.deref()，將 T 引用解為 U 的值。

```rust
use std::ops::Deref;
fn main() {
    let s = &String::from("123");
    assert_eq!(3, s.len());
    assert_eq!(3, s.deref().len());
    assert_eq!(3, (&s).len()); // s.deref().deref()
    assert_eq!(3, (&&s).len()); // s.deref().deref().deref()
    assert_eq!(3, (&&&&&&s).len()); // s.deref().deref().deref()...
}
```

### Drop trait

* 可視為物件的解構函式，當物件離開 scope 自動呼叫，可用來釋放資源。
* 為 RAII pattern 的資源管理機制。&#x20;
* Rustc 不能直接呼叫drop方法，必衝使用 `std::mem::drop`。 Rustc 不保證一定會呼叫（例如 FFI 不需要）。

```rust
fn main() {
    let v = vec![1, 2, 3];

    drop(v); // Explicitly drop.
    {
        let v = vec![1, 2, 3];
    } // Implicitly drop. Call drop(&v) when v is out of scope.
}
```

## std crate的智慧指標

`Box`、`Rc`、`Arc`、`Cel`、`RefCell`、`Mutex`、`RwLock`、`Atomic*`、`Vec`、`String` ，以及在 [std::collections](https://doc.rust-lang.org/std/collections/index.html) 的集合型別。

### 繼承可變性(Inherited mutability)

一個資料結構內部欄位的可變性取決於「變數是可變的綁定或是不變的綁定」。可<mark style="color:red;">變性一旦決定就會影響全體，每個欄位都會「繼承」相同的可變性</mark>。

```rust
fn main() {
    struct S(u8, u8);
    let mut s1 = S(1, 2);
    s1.0 += 2; // s1 is mutable, so s1.0 is also mutable.
    let s2 = S(1, 2);
    s2.0 += 2; // Compile failed. s2 is immutable.
}
```

### 內部可變性(Interior mutability)

若一個型別可以透過 shared reference 來修改其內部狀態，則我們稱之有內部可變性 ，這會透過 UnsafeCell 這個黑盒子實作。

UnsafeCell 是 safe Rust 中少數可以無視「共享不可變，可變不共享」的型別，Cell 與 RefCell 都是其衍生型別。
