# 資料處理polars

## 簡介

Polars 是一個在 Rust 中實現的極快的 DataFrames 庫，使用Apache Arrow Columnar Format作為記憶體模型。

Polars存在兩種API，一種是Eager API，另一種則是Lazy API。

* 其中Eager API和Pandas的使用類似，語法差不太多，立即執行就能產生結果。
* 而Lazy API就像Spark，首先將查詢轉換為邏輯計劃，然後對計劃進行重組最佳化，以減少執行時間和內存使用。

![polars API](.gitbook/assets/polars\_api-min.png)



## 參考資料

* [polars official site](https://www.pola.rs/)
* [polars user guide](https://pola-rs.github.io/polars-book/user-guide/index.html)
* [Eager API cookbook](https://pola-rs.github.io/polars/polars/docs/eager/index.html)
* [Lazy API cookbook](https://pola-rs.github.io/polars/polars/docs/lazy/index.html)

