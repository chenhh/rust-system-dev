# 佇列(queue)

## 簡介

佇列是一種特殊的線性表，特殊之處在於它只允許在前端（front）進行刪除操作，而在後端（rear）進行插入操作。

## 以Vec實作

```rust
#[derive(Debug)]
struct Queue<T> {
    qdata: Vec<T>,
}
impl<T> Queue<T> {
    fn new() -> Self {
        Queue { qdata: Vec::new() }
    }
    fn push(&mut self, item: T) {
        self.qdata.push(item);
    }
    fn pop(&mut self) -> Option<T> {
        if self.qdata.len() == 0 {
            return None;
        }
        Some(self.qdata.remove(0))
    }
}
fn main() {
    let mut q = Queue::new();
    q.push(1);
    q.push(2);
    println!("{:?}", q);
    q.pop();
    println!("{:?}", q);
    q.pop();
    println!("{:?}", q);
    q.pop();
    println!("{:?}", q);
}
```
