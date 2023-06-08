# 優先佇列(priority queue)

## 簡介

普通的佇列是一種先進先出的資料結構，元素在佇列尾部加入，而從佇列頭部刪除。

在優先佇列中，元素被賦予優先順序。當訪問元素時，具有最高優先順序的元素最先刪除。優先佇列具有最高級先出 （largest-in，first-out）的行為特徵。

優先佇列是0個或多個元素的集合，每個元素都有一個優先權或值，對優先佇列執行的操作有：

* 查詢;&#x20;
* 插入一個新元素;
* 刪除。

在最小優先佇列(min priority queue)中，查詢操作用來搜尋優先權最小的元素，刪除操作用來刪除該元素；對於最大優先佇列(max priority queue)，查詢操作用來搜尋優先權最大的元素，刪除操作用來刪除該元素。優先權佇列中的元素可以有相同的優先權，查詢與刪除操作可根據任意優先權進行。

```rust
#[derive(Debug)]
struct PriorityQueue<T>
where
    T: PartialOrd + Clone,
{
    pq: Vec<T>,
}

impl<T> PriorityQueue<T>
where
    T: PartialOrd + Clone,
{
    fn new() -> PriorityQueue<T> {
        PriorityQueue { pq: Vec::new() }
    }
    fn len(&self) -> usize {
        self.pq.len()
    }
    fn is_empty(&self) -> bool {
        self.pq.len() == 0
    }
    fn insert(&mut self, value: T) {
        self.pq.push(value);
    }
    fn max(&self) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        let max = self.max_index();
        Some(self.pq[max].clone())
    }
    fn min(&self) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        let min = self.min_index();
        Some(self.pq[min].clone())
    }
    fn delete_max(&mut self) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        let max = self.max_index();
        Some(self.pq.remove(max).clone())
    }
    fn delete_min(&mut self) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        let min = self.min_index();
        Some(self.pq.remove(min).clone())
    }
    fn max_index(&self) -> usize {
        let mut max = 0;
        for i in 1..self.pq.len() - 1 {
            if self.pq[max] < self.pq[i] {
                max = i;
            }
        }
        max
    }
    fn min_index(&self) -> usize {
        let mut min = 0;
        for i in 0..self.pq.len() - 1 {
            if self.pq[i] < self.pq[i + 1] {
                min = i;
            }
        }
        min
    }
}

fn test_keep_min() {
    let mut pq = PriorityQueue::new();
    pq.insert(3);
    pq.insert(2);
    pq.insert(1);
    pq.insert(4);
    assert!(pq.min().unwrap() == 1);
}
fn test_keep_max() {
    let mut pq = PriorityQueue::new();
    pq.insert(2);
    pq.insert(4);
    pq.insert(1);
    pq.insert(3);
    assert!(pq.max().unwrap() == 4);
}
fn test_is_empty() {
    let mut pq = PriorityQueue::new();
    assert!(pq.is_empty());
    pq.insert(1);
    assert!(!pq.is_empty());
}
fn test_len() {
    let mut pq = PriorityQueue::new();
    assert!(pq.len() == 0);
    pq.insert(2);
    pq.insert(4);
    pq.insert(1);
    assert!(pq.len() == 3);
}
fn test_delete_min() {
    let mut pq = PriorityQueue::new();
    pq.insert(3);
    pq.insert(2);
    pq.insert(1);
    pq.insert(4);
    assert!(pq.len() == 4);
    assert!(pq.delete_min().unwrap() == 1);
    assert!(pq.len() == 3);
}
fn test_delete_max() {
    let mut pq = PriorityQueue::new();
    pq.insert(2);
    pq.insert(10);
    pq.insert(1);
    pq.insert(6);
    pq.insert(3);
    assert!(pq.len() == 5);
    assert!(pq.delete_max().unwrap() == 10);
    assert!(pq.len() == 4);
}

fn main() {
    test_len();
    test_delete_max();
    test_delete_min();
    test_is_empty();
    test_keep_max();
    test_keep_min();
}
```
