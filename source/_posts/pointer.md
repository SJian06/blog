---
layout: post 
title: 指针初见：从二叉树中看指针用法
date: 2024-09-25 
tags: 
    - C
    - 指针
---

## 我眼里的C语言

C语言绝对是一门十分经典且知名度相当高的语言（应该是知名度最高的编程语言吧），由于Dennis Ritchie等人用它重写了unix操作系统而名声大噪，逐渐被世人所熟知。C语言是一门很接近底层的高级语言，让我印象很深的是它是纯粹的面向过程编程的编程范式,眼花缭乱的指针操作以及极其强大的性能，诸如Redis、Nginx等高性能的程序都是使用C语言写的。  
还记得我第一次学C语言的指针时，完全不理解它有啥意义，如果是大一以刷题学习C语言的前提之下，它确实意义不大，因为大多数的题目考验的是算法上的技巧，主要涉及一些数学逻辑问题，而并不会涉及到内存分配甚至是编译器的使用(我在初次学的时候压根就不知道有编译器这东西。。。)，由此可见大学课程教育之短板。  
当时的课程群有不少初学者问一些具备ACM竞赛经验的学长指针的用处，他们也只是说这个能让你写出更加高效的代码，甚至建议能用指针就尽量用指针，这些话除了让人觉得云里雾里以外并无法加深对指针的认知。直到后来我开始为主动学习C语言，才逐渐感受到指针的用处。我目前的感受就是，对于初学者来说能不用指针的地方就不要用指针，并非所有的知识都要用到，这是应试教育带来的惯性思维。滥用指针很容易造成缓冲区溢出、内存泄露等问题，可能就会在程序运行的时候突然奔溃，新手就很难发现问题在哪，找了大半天的bug，最后发现是指针解引用符号没加。。。

## 二叉树的部分代码实现

```c
// TREE.H
typedef struct node_ {
  void *data;
  BiTreeNode *left;
  BiTreeNode *right;
} BiTreeNode;

typedef struct tree_ {
  BiTreeNode *root;
  int size;
  void (*destory)(void *data);
  void (*compare)(const void *key1, const void *p2);
} BiTree;

// TREE.C
int bitree_ins_left(BiTree *tree, BiTreeNode *node, void *data) {
  BiTreeNode *new_node, **position;
  /* insert root */
  if (node == NULL) {
    if (bitree_size(tree) > 0) {
      return -1;
    }
    position = &tree->root;
  } else {
    if (node->left != NULL) {
      return -1;
    }
    position = &node->left;
  }
  if (new_node = (BiTreeNode *)malloc(sizeof(BiTreeNode)) == NULL) {
    return -1;
  }
  new_node->data = data;
  new_node->left = NULL;
  new_node->right = NULL;
  *position = new_node;
  tree->size++;
  return 0;
}
```
> 此处代码详见 《算法精解 C语言描述》 p158 稍作改动

二叉树是一种比较简单的树，主要特点就是一个节点最多拥有两个子节点，其他的特征不细说了。代码中主要是一个指向任意类型的指针和两个指向左右子树的指针，在这里主要介绍position这个指针的用法。

## 声明和初始化

C语言中的声明和初始化是完全不同的概念。在代码中

```c
BiTreeNode *new_node, **position;
```

并非仅仅是声明，还进行了初始化。如果在Java中如此声明一个Integer变量。

```java
Integer i;
```

那么这个语句就只是声明，在作用域内使用这个变量会报错，提示未初始化。而在C中，使用为初始化的变量，比如new_node，编译器并不会报错，可以编译运行程序，不过这个指针的值会是一个无法确定的值。不光光是指针类型的变量，在C语言中的基本类型，如此声明一个局部变量不赋予初始值都会随机分配一个不确定的值。  
我们可以认为编译器会进行一个默认初始化的步骤，它给局部变量 new_node和position分配了空间并初始化了一个随机值，而这个值代表着一块不确定的内存空间，所以滥用这个值会出现不可预料的错误。如果离开一个块作用域(前提是定义在这个作用域中)，这两个变量就会自动删除，但他们所指向的空间可能就不会删除，造成了内存泄露问题。C++中提出了RAII的解决思路,而C中既不存在析构函数，也没有智能指针，只能通过手工去释放对应空间的地址，所以要求开发者具备较高的编程素养。

## 指针的指针

position 是个指针的指针，它的定义和初始化看起来比较绕。

```c
position = &node->left;
```

如此就赋予了left指针的地址给position对象，*position就代表了left指针，下面这张图可以很好地概括这个意思。

![指针的指针](img/ptr.PNG)  

```c
*position = (BiTreeNode *)malloc(sizeof(BiTreeNode));
// 访问数据
(*position)->data;
(*position)->left;
```

使用position统一表示需要插入的节点位置，然后将position指向的指针赋予新的节点地址，将新节点插入到树中。如此一来，position代表了 tree->root 或者是 node->left ，简化了部分代码。
