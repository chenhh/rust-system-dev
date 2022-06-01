# 容器與迭代器

## 常用的容器

| 名稱         | 描述                         |
| ---------- | -------------------------- |
| Vec        | 可變長度陣列，連續儲存資料              |
| VecDeque   | 雙向佇列，適用於從頭部與尾部插入與刪除資料      |
| LinkedList | 雙向鏈結，非連續儲存資料               |
| HashMap    | 以hash算法儲存key-value對        |
| BTreeMap   | 以Btree算法儲存key-value對       |
| HashSet    | 基於Hash算法的集合，相當於沒有值的HashMap |
| BTreeSet   | 基於BTree的集合 相當於沒有值的BTreeMap |
| BinaryHeap | 基於二元樹堆積實現的優先佇列             |

## 迭代器 (iterator)

Rust的迭代器是指實現了`Iterator` trait的類型。

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ...
}
```

它最主要的一個方法就是`next()`，返回一個`Option<Item>`。一般情況返回`Some(Item)`；如果迭代運算完成，就返回`None`。

```rust
use std::iter::Iterator;
// 自訂的類別
struct Seq {
    current: i32,
}
impl Seq {
    fn new() -> Self {
        Seq { current: 0 }
    }
}
// 實現迭代器
impl Iterator for Seq {
    type Item = i32;
    fn next(&mut self) -> Option<i32> {
        if self.current < 100 {
            self.current += 1;
            return Some(self.current);
        } else {
            return None;
        }
    }
}
fn main() {
    let mut seq = Seq::new();
    while let Some(i) = seq.next() {
        println!("{}", i);
    }
}
```

## 迭代器的組合

Rust標準庫有一個命名規範，從容器創造出迭代器一般有三種方法

* `iter()`創造一個Item是`&T`類型的迭代器；
* `iter_mut()`創造一個Item是`&mut T`類型的迭代器；
* `into_iter()`創造一個Item是`T`類型的迭代器(move所有權)。

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    // Item為&T的iterator
    let mut iter = v.iter();
    while let Some(i) = iter.next() {
        println!("{i}");
    }
}
```

如果迭代器就是這麼簡單，那麼它的用處基本就不大了。<mark style="background-color:red;">Rust的迭代器有一個重要特點，那它就是可組合的（composability）</mark>。

Iterator trait裡面還有一大堆的方法，比如`nth`、`map`、`filter`、`skip_while`、`take`等等，這些方法都有預設實現，它們可以統稱為adapters（適配器）。它們有個共性，返回的是一個具體類型，而這個類型本身也實現了`Iterator` trait。這意味著，我們調用這些方法可以從一個迭代器創造出一個新的迭代器。

```rust
// 迭代器寫法，容易平行化
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9];
    let mut iter = v
        .iter()
        .take(5)    // 取前5個元素
        .filter(|&x| x % 2 == 0)    // 取偶數值
        .map(|&x| x * x)    // 平方
        .enumerate();
    while let Some((i, v)) = iter.next() {
        println!("{} {}", i, v);
    }
}

// 傳統等價寫法
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9];
    let mut iter = v.iter();
    let mut count = 0;
    let mut index = 0;
    while let Some(i) = iter.next() {
        if count < 5 {
            count += 1;
            if (*i) % 2 == 0 {
                let s = (*i) * (*i);
                println!("{} {}", index, s);
                index += 1;
            }
        } else {
            break;
        }
    }
}
```

。兩個版本相比較，迭代器的可讀性是不言而喻的。這種抽象相比于直接在傳統的迴圈內部寫各種邏輯是有優勢的，特別是如果我們想把迭代器改成並存執行是非常容易的事情。而傳統的寫法涉及細節太多，不太容易改成並存執行。

分析迭代器的實現原理我們也可以知道：<mark style="background-color:red;">**構造一個迭代器本身，是代價很小的行為，因為它只是初始化了一個物件，並不真正產生或消費資料**</mark>**。**不論迭代器內部嵌套了多少層，最終消費資料還是要通過調用next（）方法實現的。這個特點，也被稱為<mark style="background-color:red;">惰性求值（lazy evaluation）</mark>。

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    // 什麼事都沒做，因為map只是iterator
    // 沒有真正操作內容資料
    let data = v.iter().map( |x| println!("{}", x));
    // 要真正走訪iterator才會執行
    for _ in data{
    }
}
```

### [\[rust中文社區\] 迭代器是如何做到避免邊界檢查的](https://rustcc.cn/article?id=295bb1bf-d826-4da0-be02-a721052ab901)？

迭代器不是避免邊界檢查，是避免額外的邊界檢查。

{% code title="" %}
```rust
for i in 0..length{
    arr[i] = 0;
}
```
{% endcode %}

由於arr的長度並不一定真的是length，如果arr不是length長度，那程式就會越界訪問。而rust要求安全，在length並不等於arr長度的時候，也要能導致程式panic，所以在arr\[i]時也要進行一次邊界檢查來檢視i是否超出arr的長度。 這樣在rust裡就變成了對for循環變量是否到達length進行一次檢查，然後對循環變量索引是否對於arr越界進行一次邊界檢查，就有了兩次檢查，如果我們能保證length就是arr的length，那這次多餘的檢查就是無必要的。

那迭代器是怎麼避免的，rust迭代器是這麼寫的：

```rust
for e in arr.iter_mut(){
    *e = 0;
}
```

因為迭代器是由arr創建的，那麼就能保證只要在next中進行一次檢查，如果返回的是Some(\&e)就是必然有效的，那麼第二次邊界檢查就可以省掉。 當然，第一種寫法是可能被編譯器最佳化掉的，例如編譯器可以判定length就是arr的長度，不過這在代碼復雜的時候並不夠可靠，所以還是需要用迭代器來保證進行這種最佳化，這也是官方推薦使用迭代器來避免邊界檢查的原因。

## for循環

Rust裡面更簡潔、更自然地使用迭代器的方式是使用for迴圈。本質上來說，for迴圈就是專門為迭代器設計的一個語法糖。for迴圈可以對針對陣列切片、字串、Range、Vec、LinkedList、HashMap、BTreeMap等所有具有迭代器的類型執行迴圈，而且還允許我們針對自訂類型實現迴圈。

```rust
use std::collections::HashMap;
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9];
    // borrow
    for i in &v {
        print!("{i}\t");
    }
    println!("");
    let map: HashMap<i32, char> = [(1, 'a'), (2, 'b'), (3, 'c')].iter().cloned().collect();
    // borrow
    for (k, v) in &map {
        println!("{k} : {v}");
    }
}
```

```rust
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```

只要某個類型實現了`IntoIterator`，那麼調用`into_iter()`方法就可以得到對應的迭代器。這個`into_iter()`方法的receiver是`self`，而不是`&self`，執行的是move語義。這麼做，可以同時支援Item類型為`T`、`&T`或者`&mut T`，用戶有選擇的權力。

```rust
impl<K, V> IntoIterator for BTreeMap<K, V> {
    type Item = (K, V);
    type IntoIter = IntoIter<K, V>;
}
impl<'a, K: 'a, V: 'a> IntoIterator for &'a BTreeMap<K, V> {
    type Item = (&'a K, &'a V);
    type IntoIter = Iter<'a, K, V>;
}
impl<'a, K: 'a, V: 'a> IntoIterator for &'a mut BTreeMap<K, V> {
    type Item = (&'a K, &'a mut V);
    type IntoIter = IterMut<'a, K, V>;
}
```

對於一個容器類型，標準庫裡面對它impl了三次IntoIterator。當Self類型為BTreeMap的時候，Item類型為（K，V），這意味著，每次next()方法都是把內部的元素move出來了；當Self類型為\&BTreeMap的時候，Item類型為（\&K，\&V），每次next()方法返回的是借用；當Self類型為\&mut BTreeMap的時候，Item類型為（\&K，\&mut V），每次next()方法返回的key是唯讀的，value是可讀寫的。

Rust的for\<item>in\<container>{\<body>}語法結構就是一個語法糖。這個語法的原理其實就是調用**`<container>.into_iter()`**方法來獲得迭代器，然後不斷迴圈調用迭代器的`next()`方法，將返回值解包，賦值給\<item>，然後調用\<body>語句塊。

在使用for迴圈的時候，我們可以自主選擇三種使用方式：

```rust
// container在迴圈之後生命週期就結束了,
// 迴圈過程中的每個item是從container中move出來的
for item in container {}
// 迭代器中只包含container的&型引用,
// 迴圈過程中的每個item都是container中元素的借用
for item in &container {}
// 迭代器中包含container的&mut型引用,
// 迴圈過程中的每個item都是指向container中元素的可變借用
for item in &mut container{}
```

Rust的IntoIterator trait實際上就是for語法的擴展介面。如果我們需要讓各種自訂容器也能在for迴圈中使用，那就可以借鑒標準庫中的寫法，自行實現這個trait即可。

