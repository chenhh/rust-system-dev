# FFI

## 簡介

"FFI"是" Foreign Function Interface"的縮寫，大意為不同程式語言所寫程式間的相互調用。

每一種程式語言都可能定義許多“基礎類型”， 兩種程式語言的基礎類型之間最好有一個交集， 這樣才能傳遞資料， 所以：`Rust std::ffi` 和 The libc crate就是非常重要的C and Rust的基礎類型交集，

## Rust支援的函式庫類型

* lib — Generates a library kind preferred by the compiler, currently defaults to rlib.&#x20;
* rlib — A Rust static library.&#x20;
* staticlib — A native static library. (與C ABI相容)
* dylib — A Rust dynamic library.&#x20;
* cdylib — A native dynamic library.  (與C ABI相容)
* bin — A runnable executable program.&#x20;
* proc-macro — Generates a format suitable for a procedural macro library that may be loaded by the compiler.

## rust-bindgen

\[[github](https://github.com/rust-lang/rust-bindgen)]

這個專案能夠為你的表頭檔案以及相關結構體自動生成對接的rust程式碼， 不需要手動去寫 一些函數，目前的缺點是不能夠處理C++的虛函數。

Rust對於c++的程式碼其實只是有限的對接，並不能百分百的封裝c++的介面， 因為有些特性不一樣，但是Rust對c的介面支援非常非常的友好， 所以我們可以封裝c的介面來封裝c++的介面。

## 參考資料

