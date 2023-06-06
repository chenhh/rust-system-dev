---
description: raw pointer
---

# 裸指標(原始指標)

## 簡介

_`*const T`和`*`_`mut T`在Rust中被稱為「裸指標」。它允許別名，允許用來寫共享所有權的類型，甚至是記憶體安全的共用記憶體類型如：Rc和Arc，但是賦予你更多權利的同時意味著你需要擔當更多的責任：

* 不能保證指向有效的記憶體，甚至不能保證是非空的。
* 沒有任何自動清除，所以需要手動管理資源。
* 是普通舊式類型，也就是說，它不移動所有權，因此Rust編譯器不能保證不出像釋放後使用這種bug。
* 缺少任何形式的生命週期，不像&，因此編譯器不能判斷出懸垂指針。
* 除了不允許直接通過\*const T改變外，沒有別名或可變性的保障。

## 從引用建立一個裸指標

```rust
fn main() {
    let a = 1;
    //將引用轉成裸指標是安全的操作
    let b = &a as *const i32;
    let mut x = 2;
    //將引用轉成裸指標是安全的操作
    let y = &mut x as *mut i32;
    // 解開裸指標是不安全的操作，必須在unsafe區塊
    unsafe {
        println!("b={}, y={}", *b, *y);
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
