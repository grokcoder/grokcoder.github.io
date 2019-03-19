---
layout: post
title: Rust学习笔记之内存管理
date: 2019-03-19
author: bitking
tags: [coding, rust]
feature-img: "assets/img/pexels/computer.jpeg"
---

> Rust语言的安全性很大部分来源于其对变量内存的精细化管理，Rust的内存管理技术在GC类语言（eg:Java）和手动释放类语言(eg:C++/C)之间. Rust的所有变量的分布整体上呈现成树状结构，每个变量都有一个唯一的拥有者，这个拥有者的生命周期决定了变量存活的生命周期。这种方式为变量的内存释放和前期安全性检查提供了遍历，但是也同时为编码提出了很多限制。


## 1.ownership

### 变量内存排布及回收原则

“用户决定每个变量的生命周期，Rust回收变量的内存以及其他相关资源。” -- 这是Rust内存管理的最高原则。在Rust中每个value都只有一个单一的owner，只有这个owner决定了该值的生命周期。当owner被dropped他的value也会随着被dropped。当变量离开了它的影响返回，它就会被自动回收。下面结合几个常见变量的生命周期进行分析：


* vector 容器类型的内存分配，

```rust
fn print_padovan() {
    let mut padovan = vec![1, 1, 1];// allocated here
    for i in 3 .. 10 {
        let next = padovan[i - 3] + padovan[i - 2]
        padovan.push(next);
    }
    print!("P(1..10)= {:?}", padovan); // dropped here
}
```

padovan 的内存分配如下, vec类型的变量分配在栈上，其存储的内容分配在堆上，栈上分配内容包括:1. 指正，2. vec的容量大小， 3. vec 当前存储的元素长度

![vec_allocate](http://cdn.bitking.wang/2019-03-19-vec_allocate.png)



* Box 类型变量的内存分配

普通简单变量直接分配在栈上，通过Box封装的变量存储在堆上，如下例子所示:

```rust
{
    let point = Box::new((0.625, 0.5));// point allocated here
    let label = format!(":?", point); // label allocated here
    assert_eq!(label, "(0.625, 0.5)") // both dropped here    
}
```

![box_allocate](http://cdn.bitking.wang/2019-03-19-box_allocate.png)


* Rust的复杂引用关系会形成一棵ownership树

```rust
    struct Person {name: String, birth: i32}
    
    let mut composers = Vec::new();

    composers.push(Person{name: "Palestrina".to_string(), birth: 1525});
    composers.push(Person{name: "Downland".to_string(), birth: 1563});
    composers.push(Person{name: "Lully".to_string(), birth: 1632});


    for composer in &composers {
        println!("{}, born {}", composer.name, composer.birth)
    }
```

 在该例子中，变量composers 指向一个vector, vector中的元素是一个结构体，它指向具体结构体的内存结构(参考string类型内存结构和i32类型存储结构)。

![complex_tree](http://cdn.bitking.wang/2019-03-19-complex_tree.png)


Rust的ownership 机制总结：
1. 你可以将一个变量从一个owner转移到另一个owner;
2. 标准库中提供了`Rc` 和 `Arc`， 这些允许某些值同时拥有多个owners；
3. 你可以使用引用（ **borrow a reference** )，但引用是非拥有指针，具有有限的生命周期


### move - 变量所属变更机制

在Rust中类似变量赋值、参数传递、或者函数返回值都不会赋值value, 而是采用一种`move`的机制。

`move`机制：原owner放弃value的拥有权并转交给destination,并且自身变成未初始化状态；之后destination控制value的生命周期。

> ps: 针对`Copy types` 都采用复制的方式，默认的 `Copy types`有：整型、浮点型、字符型、布尔类型，`Copy types`组成的元组以及固定长度的数组也是`Copy tupes`


我们来看一个简单的赋值的例子：

```rust
    let s = vec!["undon".to_string(), "ramen".to_string(), "soba".to_string()]
    let t = s;
    let u = s; // error: use of moved value 's'
```

`t = s` 执行之前：

![assignment_before](http://cdn.bitking.wang/2019-03-19-assignment_before.png)


`t = s` 执行之后：

![assignment_afte](http://cdn.bitking.wang/2019-03-19-assignment_after.png)



ps: 
> `Java` 采用可达性分析的方式确定变量的生命周期，复制的直接是引用；`Python`中采用引用技术法 复制的也是引用，但是会记录原始变量呗引用的数量；`C++` 的赋值操作会直接赋值整个value，会触发相应的复制构造函数，因此开销比较大。相比而言`Rust`采用 `move`的机制效率较高，内存管理相对简单，但是统一时间单一变量仅有一个可用引用。


由于使用了`move` 的机制，当我们直接使用集合内部元素的时候就会发生控制权的转移，因此为了防止集合内部出现未初始化的变量，rust不允许直接使用集合内部元素进行复制，rust推荐使用引用的方式进行引用相应变量。

还是之前的例子：

```rust

let s = vec!["udon".to_string(), "ramen".to_string(), "soba".to_string()];

let t = s[0]; // error[E0507]: cannot move out of borrowed content
    
let m = &[0]; // ok
```


### 共享ownership - `Rc` 和 `Arc`

`Rc`和`Arc`: 引用技术类型指针，当某些变量需要同时被多个分支使用，且需要在所有使用者使用之后再进行释放的场景。

其中 `Arc` 是线程安全的，Atomic reference count 
而 `Rc` 则是非线程安全。`Rc`的实现原理是其指向的变量的内容在堆上分配，并且堆上还记录了该变量的引用计数，但是Rc指向的变量必须是不可变。

```rust
let s: Rc<String> = Rc::new("shirataki".to_string());

let k: Rc<String> = s.clone();
let u = s.clone();
```

![r](http://cdn.bitking.wang/2019-03-19-rc.png)


## 2.引用

引用在Rust中被称为非拥有性（nonowning pointer）指针，这种指针对所指向的对象的生命周期没有影响。

`Rust` 中创建某个指针也称为borrowing the value: what you have borrowed, you must eventually return to its owner.

引用分两种：

* **共享引用**: 只能读取被引用的对象，当某个对象有共享引用时，即使是该对象的owner 也不可以修改对象。

```rust
  let t = &T;
```
* **可变引用**: 可以修改被引用的对象内容，可变引用和共享引用是互斥的。

```rust
let t = &mut T;
```
对一个引用的集合的遍历，也会导致循环内部的集合元素的的访问变成相应的引用的方式。

### 通过引用访问原始值

在Rust中通过引用访问原始值，标准语法需要使用 `*` 操作符，但是由于引用在Rust中运用广泛，`.` 操作符也可以做隐式的变量的原始值的获取。

* Rust 中支持引用赋值，相当于更换指向的位置；
* 通过 `==` 比较的是引用的原始值；
* 引用永远不可以为Null;

### 引用安全性

Rust中使用引用的时候需要保证引用和其所引用的变量的生命周期是否匹配，需要保证引用的生命周期应该在所引用的变量的生命周期范围内。

可以使用生命周期的限制符限制函数参数或者返回值引用的生命周期范围。
如下所示，<'a> 用来限制所传入函数f的参数p的生命周期应该是局限在函数内部的，也就是说f 中不应该让p赋值给生命周期超过函数的返回的变量。

```rust
fn f<'a> (p: &'a i32) {
    // <'a> 表示一个f的生命周期参数
}
```

如下所示限制了函数的返回值引用的生命周期，表示返回值的生命周期和传入参数的生命周期一致。

```rust
fn smallest<'a>(v: &'a [i32]) -> &'a i32 {

}
```

如下是限制结构体的内变量的生命周期，表示结构体内部r引用的变量的生命周期应该和S保持一致。

```rust
struct S<'a> {
    r: &'a i32
}
```