---
description: raw pointer
---

# 原始指標

## 簡介

Rust 在標準庫有許多不同的智慧指標類型，但是有兩種特別的類型。Rust 的安全來自於編譯時檢查，但原始指標沒有這樣的保證，使用起來不安全。

Rust 支援兩種原始指標：

* 不可變原始指標 `*const T` 。
* 可變原始指標 `*&mut T`。 　　
* <mark style="color:red;">在執行階段，一個原始指標 \* 和一個指向同一塊資料的引用具有相同的表示</mark>。　　

有時，當寫庫的某些類型時，出於某種原因你需要繞過 Rust 的安全保證。在這種情況下，你可以使用原始指標來實現你的庫，同時為給使用者一個安全介面。

原始指標與其他指標類型是不同的地方：

* 不能保證指向有效的記憶體，甚至不能保證是非空的(Box和&可保證非空)。
* 沒有任何自動清除，所以需要手動管理資源。
* 是普通舊式類型，也就是說，它不移動所有權，因此Rust編譯器不能保證不出像釋放後使用這種bug。
* 缺少任何形式的生命週期，不像&，因此編譯器不能判斷出懸垂指針。
* 除了不允許直接通過\*const T改變外，沒有別名或可變性的保障。

用 as 運算子可以將引用轉為原始指標，也可直接在變數宣告時指定類型：

```rust
fn main() {
    let mut x = 10;
 
    // 使用as轉型，建立一個原始指標是絕對安全的
    let ptr_x = &mut x as *mut i32;
    // 用類型宣告建立原始指三
    // 如果是引用，只能在區塊中有一個mut，但原始指標無此限制
    let ptr_x2: *mut i32 = &mut x;
    
    // 建立一個原始指標是絕對安全的
    let y = Box::new(20);
    let ptr_y = &*y as *const i32;

    // 原生指標操作與取值*要放在unsafe中執行
    unsafe {
        *ptr_x += *ptr_y;
        println!("value of ptr_x:{}", *ptr_x); //30
        *ptr_x2 += *ptr_y;
        println!("value of ptr_x2:{}", *ptr_x2); //50
    }
}
```

## 從引用建立一個原始指標

```rust
fn main() {
    let a = 1;
    //將引用轉成裸指標是安全的操作
    let b = &a as *const i32;
    let a_ref = &a; // reference

    let mut x = 2;
    //將引用轉成裸指標是安全的操作
    let y = &mut x as *mut i32;

    // 引用可在一般區塊使用
    println!("a_ref={}", *a_ref); // 1
    // 解開裸指標是不安全的操作，必須在unsafe區塊                             
    unsafe {
        println!("b={}, y={}", *b, *y); // 1, 2
    }
}

```

## Box的into\_raw

引用和裸指標之間可以隱式轉換，但隱式轉換後再解引用需要使用unsafe。

```rust
fn main() {
    let a: Box<i32> = Box::new(10);
    // 我們需要先解引用a，再隱式把 & 轉換成 *
    let b: *const i32 = &*a;
    // 使用 into_raw 方法
    let c: *const i32 = Box::into_raw(a);

    // 顯式
    let a = 1;
    let b: *const i32 = &a as *const i32;
    //或者let b = &a as *const i32；
    // 隱式
    let c: *const i32 = &a;
    unsafe {
        println!("{}", *c);
    }
}
```
