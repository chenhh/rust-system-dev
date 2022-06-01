# 斷言(assert)

## 簡介

Rust支援6種斷言。分別是：

* `assert!`：用於斷言布林表達式是否為true。
* `assert_eq!`：用於斷言兩個表達式是否相等。
* `assert_ne!`：用於斷言兩個表達式是否不相等。
* debug\_assert!：同上，但只用於debug模式。
* debug\_assert\_eq!
* debug\_assert\_ne!

從名稱我們就可以看出來這6種斷言，可以分為兩大類，帶debug的和不帶debug的，它們的區別就是assert開頭的在除錯模式和發佈模式下都可以使用，而debug開頭的只可以在除錯模式下使用。當不符合條件時，斷言會引發`panic!`。

Rust處理異常的方法有4種：Option、Result、線程恐慌（Panic）、程式終止（Abort）。

