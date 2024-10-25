# use 和 extern crate 的區別

extern crate foo 表示你想要連結到一個外部庫，並將頂級 crate 名稱帶入範疇（相當於 use foo）。從 Rust 2018 開始，在大多數情況下，你不再需要使用 extern crate，因為 Cargo 會通知編譯器存在哪些 crate。

extern crate簡單的說是 Rust 引入庫的一種方式，但現在除了少數情況，並不主張顯式使用，只要使用use crate即可。

## 參考資料

[https://rustcc.cn/article?id=71892109-426e-4cb8-a9bd-94882231fc00](https://rustcc.cn/article?id=71892109-426e-4cb8-a9bd-94882231fc00)
