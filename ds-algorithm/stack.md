# 堆疊(stack)

## 簡介

堆疊作為一種資料結構，是一種只能在一端進行插入和刪除操作的特殊線性表。

它按照先進後出的原則儲存資料，先進入的資料被壓入底部，最後的資料在頂部，需要讀資料的時候從堆疊頂開始彈出資料（最後一個資料被第一個讀出來）。

## 以link list實作

```rust
#[derive(Debug)]
struct Stack<T> {
    top: Option<Box<StackNode<T>>>,
}

#[derive(Clone, Debug)]
struct StackNode<T> {
    val: T,
    // next為None時，為bottom
    next: Option<Box<StackNode<T>>>,
}

impl<T> StackNode<T> {
    fn new(val: T) -> StackNode<T> {
        StackNode {
            val: val,
            next: None,
        }
    }
}

impl<T> Stack<T> {
    fn new() -> Stack<T> {
        Stack { top: None }
    }
    fn push(&mut self, val: T) {
        let mut node = StackNode::new(val);
        let next = self.top.take();
        node.next = next;
        self.top = Some(Box::new(node));
    }
    fn pop(&mut self) -> Option<T> {
        let val = self.top.take();
        match val {
            None => None,
            Some(mut x) => {
                self.top = x.next.take();
                Some(x.val)
            }
        }
    }
}
fn main() {
    #[derive(PartialEq, Eq, Debug)]
    struct TestStruct {
        a: i32,
    }
    let a = TestStruct { a: 5 };
    let b = TestStruct { a: 9 };
    let mut s = Stack::<&TestStruct>::new();
    assert_eq!(s.pop(), None);
    s.push(&a);
    s.push(&b);
    println!("{:?}", s);
    assert_eq!(s.pop(), Some(&b));
    assert_eq!(s.pop(), Some(&a));
    assert_eq!(s.pop(), None);
}
u
```
