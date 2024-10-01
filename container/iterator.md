# 迭代器

在Rust中，迭代器共分為三個部分：<mark style="background-color:red;">迭代器(iterator)、介面卡(adapter)、消費者(consumer)</mark>。

其中，迭代器本身提供了一個惰性的序列，介面卡對這個序列進行諸如篩選、拼接、轉換尋找等操作，消費者則在前兩者的基礎上生成最後的數值集合。

消費者是迭代器上一種特殊的操作，其主要作用就是將迭代器轉換成其他類型的值，而非另一個迭代器。

而介面卡，則是對迭代器進行遍歷，並且其生成的結果是另一個迭代器，可以被鏈式呼叫直接呼叫下去。

Rust的迭代器是指實現了[`Iterator` trait](https://doc.rust-lang.org/std/iter/index.html)的類型。

```rust
trait Iterator {
    type Item;    // 關聯類型
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
// 建構子
impl Seq {
    fn new() -> Self {
        Seq { current: 0 }
    }
}
// 實現迭代器
impl Iterator for Seq {
    type Item = i32; // 關聯類型
    fn next(&mut self) -> Option<i32> {
        if self.current < 100 {
            self.current += 1;
            Some(self.current)
        } else {
            None
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

Rust標準庫有一個命名規範，從容器創造出迭代器一般有三種方法：

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

`Iterator` trait裡面還有一大堆的方法，比如`nth`、`map`、`filter`、`skip_while`、`take`等等，這些方法都有預設實現，它們可以統稱為adapters（適配器）。它們有個共性，返回的是一個具體類型，而這個類型本身也實現了`Iterator` trait。這意味著，我們調用這些方法可以從一個迭代器創造出一個新的迭代器。

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

兩個版本相比較，迭代器的可讀性是不言而喻的。這種抽象相比于直接在傳統的迴圈內部寫各種邏輯是有優勢的，特別是如果我們想把迭代器改成並存執行是非常容易的事情。而傳統的寫法涉及細節太多，不太容易改成並存執行。

分析迭代器的實現原理我們也可以知道：<mark style="background-color:red;">**構造一個迭代器本身，是代價很小的行為，因為它只是初始化了一個物件，並不真正產生或消費資料**</mark>。不論迭代器內部嵌套了多少層，最終消費資料還是要通過調用`next（）`方法實現的。這個特點，也被稱為<mark style="background-color:red;">惰性求值（lazy evaluation）</mark>。

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

Rust裡面更簡潔、更自然地使用迭代器的方式是使用for迴圈。for循環能夠依次對迭代器的任意元素進行訪問，即for迴圈就是專門為迭代器設計的一個語法糖。

for迴圈可以對針對陣列切片、字串、Range、Vec、LinkedList、HashMap、BTreeMap等所有具有迭代器的類型執行迴圈，而且還允許我們針對自訂類型實現迴圈。

```rust
use std::collections::HashMap;
fn main() {
    // 1..10 Range為迭代器，呼叫next()走訪元素
    for i in 1..10 {
        print!("{} ", i);
    }

    let v = vec![1, 2, 3, 4, 5, 6, 7, 8, 9];
    // borrow of vec
    // Vec沒有實現Iterator，而是實現IntoIterator
    for i in &v {
        print!("{i}\t");
    }
    println!("");
    let map: HashMap<i32, char> = [(1, 'a'), (2, 'b'), (3, 'c')].iter().cloned().collect();
    // borrow of map
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

Rust的for\<item>in\<container>{\<body>}語法結構就是一個語法糖。這個語法的原理其實就是調用`<container>.into_iter()`方法來獲得迭代器，然後不斷迴圈調用迭代器的`next()`方法，將返回值解包，賦值給\<item>，然後調用\<body>語句塊。

### 在使用for迴圈的時候，我們可以自主選擇三種使用方式

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

Rust的IntoIterator trait實際上就是for語法的擴展介面。如果我們需要讓各種自訂容器也能在for迴圈中使用，那就可以借鑒標準庫中的寫法，自行實現這個特徵即可。

## 無限迭代器

```rust
let inf_seq = (1..).into_iter();
```

不用擔心這個無限增長的序列撐爆你的記憶體，因為介面卡的惰性求值的特性，它本身是安全的，除非你對這個序列進行collect或者fold。

想要應用這個迭代號，你需要用take或者take\_while來截斷它。

## 消費者

迭代器負責生產，而消費者則負責將生產出來的東西最終做一個轉化。

取第幾個值所用的 `.nth()`函數，還有用來尋找值的 `.find()` 函數，呼叫下一個值的`next()`函數。

### collect方法

一個典型的消費者就是collect。此方法負責將迭代器裡面的所有資料取出。

```rust
fn main() {
    // 解法1，手動指定v的容器, 類型_讓編譯器自動判斷
    let v: Vec<_> = (1..20).collect();
    println!("{:?}", v);
    
    // 解法2，顯式地指定collect呼叫時的類型
     let v2 = (1..20).collect::<Vec<_>>();
     println!("{:?}", v2);
}
```

collect只知道將迭代器收集到一個實現了 FromIterator 的類型中去，但是，事實上實現這個 trait 的類型有很多（Vec, HashMap等），因此，collect沒有一個上下文來判斷應該將v按照什麼樣的方式收集，必須手動指定類型。

### fold方法

fold函數即為map-reduce中的reduce函數。

```rust
// fold(base, |accumulator, element| .. )
fn main() {
    let m = (1..20).fold(0, |add, x| add + x);
    println!("m={m}"); // 190
}
```

fold的輸出結果的類型，最終是和base的類型是一致的（如果base的類型沒指定，那麼可以根據前面m的類型進行反推，除非m的類型也未指定）。

## 介面卡

生產消費者的模型裡，生產者所生產的東西不一定都會被消費者買賬，因此，需要對原有的產品進行再組裝。這個再組裝的過程，就是介面卡。因為介面卡返回的是一個新的迭代器，所以可以直接用鏈式請求一直寫下去。

### map方法

[map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map)接受一個閉包函數，對迭代器的每一個元素使用此函數。

```rust
fn main() {
    // map為lazy eval，不會實際動作，必須有consumer才能編譯
    let m: Vec<_> = (1..5).map(|x| x + 1).collect();
    println!("{:?}", m); // [2, 3, 4, 5]
}
```

### filter方法

[filter](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter)接受一個閉包涵數，返回一個布林值，返回true的時候表示保留元素，false丟棄之。

```rust
fn main() {
    // filter為lazy eval，不會實際動作，必須有consumer才能編譯
    let v: Vec<_> = (1..20).filter(|x| x % 2 == 0).collect();
    println!("{:?}", v); // [2, 4, 6, 8, 10, 12, 14, 16, 18]
}
```

### skip和take方法

[take(n)](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.take)的作用是取前n個元素，而[skip(n)](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.skip)正好相反，跳過前n個元素。

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];
    let v_take = v.iter().cloned().take(2).collect::<Vec<_>>();
    assert_eq!(v_take, vec![1, 2]);

    let v_skip: Vec<_> = v.iter().cloned().skip(2).collect();
    assert_eq!(v_skip, vec![3, 4, 5, 6]);
}
```

### zip與unzip

[zip](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip)是一個介面卡，他的作用就是將兩個迭代器的內容壓縮到一起，形成 Iterator\<Item=(ValueFromA, ValueFromB)>

```rust
use std::collections::HashMap;

fn main() {
    let names = vec!["WaySLOG", "Mike", "Elton"];
    let scores = vec![60, 80, 100];
    let score_map: HashMap<_, _> = names.iter().zip(scores.iter()).collect();
    println!("{:?}", score_map);
    // {"Elton": 100, "WaySLOG": 60, "Mike": 80}
}
```

[unzip](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.unzip)是將zip後的元素解開。

```rust
fn main() {
    let a = [(1, 2), (3, 4), (5, 6)];
    // 第一個元素解開成left vec，第二個元素解開成right vec
    let (left, right): (Vec<_>, Vec<_>) = a.iter().cloned().unzip();

    assert_eq!(left, [1, 3, 5]);
    assert_eq!(right, [2, 4, 6]);

    // you can also unzip multiple nested tuples at once
    let a = [(1, (2, 3)), (4, (5, 6))];

    let (x, (y, z)): (Vec<_>, (Vec<_>, Vec<_>)) = a.iter().cloned().unzip();
    assert_eq!(x, [1, 4]);
    assert_eq!(y, [2, 5]);
    assert_eq!(z, [3, 6]);
}
```

### enumerate

[enumerate](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.enumerate)把迭代器的下標顯示出來。

```rust
fn main() {
    let v = vec![1u64, 2, 3, 4, 5, 6];
    let val = v
        .iter()
        .enumerate()
        // 迭代生成標，並且每兩個元素剔除一個
        .filter(|&(idx, _)| idx % 2 == 0)
        // 將下標去除,如果呼叫unzip獲得最後結果的話，可以呼叫下面這句，終止鏈式呼叫
        // .unzip::<_,_, vec<_>, vec<_>>().1
        .map(|(idx, val)| val)
        // 累加 1+3+5 = 9
        .fold(0u64, |sum, acm| sum + acm);
    println!("{}", val); // 9
}
```

### find與position

[find](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.find)傳入一個閉包函數，從開頭到結尾依次尋找能令這個閉包返回true的第一個元素，返回Option\<Item>。

```rust
fn main() {
    let a = [1, 2, 3];

    assert_eq!(a.iter().find(|&&x| x == 2), Some(&2));
    assert_eq!(a.iter().find(|&&x| x == 5), None);

    let a = [1, 2, 3];
    let mut iter = a.iter();
    assert_eq!(iter.find(|&&x| x == 2), Some(&2));
    // we can still use `iter`, as there are more elements.
    assert_eq!(iter.next(), Some(&3));
}
```

[position()](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.position): 類似find函數，不過這次輸出的是Option，第幾個元素。

```rust
fn main() {
    let a = [1, 2, 3];

    assert_eq!(a.iter().position(|&x| x == 2), Some(1));
    assert_eq!(a.iter().position(|&x| x == 5), None);

    let a = [1, 2, 3, 4];

    let mut iter = a.iter();
    assert_eq!(iter.position(|&x| x >= 2), Some(1));
    // we can still use `iter`, as there are more elements.
    assert_eq!(iter.next(), Some(&3));
    // The returned index depends on iterator state
    assert_eq!(iter.position(|&x| x == 4), Some(0));
}
```

### all與any

[all](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.all)傳入一個函數，如果對於迭代器內的任意一個元素，呼叫這個函數返回false，則整個表示式返回false；否則返回true，即所有元素使用函數都為真。

[any](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.any)也是傳入一個函數，不過這次是任何一個元素返回true，則整個表示式返回true，否則false。

```rust
fn main() {
    let a = [1, 2, 3];

    assert!(a.iter().all(|&x| x > 0));

    assert!(!a.iter().all(|&x| x > 2));

    let a = [1, 2, 3];

    let mut iter = a.iter();
    // 在第一個傳回false的pos停止
    assert!(!iter.all(|&x| x != 2));
    // we can still use `iter`, as there are more elements.
    assert_eq!(iter.next(), Some(&3));
}
r
```

### max和min

尋找整個迭代器裡所有元素，返回最大或最小值的元素。注意作用在浮點數上會有不符合預期的結果。
