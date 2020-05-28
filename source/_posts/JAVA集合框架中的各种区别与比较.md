---
title: JAVA集合框架中的各种区别与比较
date: 2019-01-24 21:22:21
tags: 集合
categories: JAVA基础
---
## List Set Map 的区别

- **List： **有序，可以多个元素引用相同的对象
- **Set： **无序，不重复，不可以多个元素引用相同对象
- **Map： **使用键值对存储，两个key可以引用相同的对象，但是key不能重复

## ArrayList 和LinkedList 区别

- **ArrayList： ** 底层使用数组，存、读效率高；插入、删除特定位置效率低，近似O(n)
- **LinkedList： **使用双向链表，插入、删除效率高O(1)

## ArrayList 和 Vector 区别

- Vector的所有方法都是同步的，ArrayList不同步
- 由于Vector类的方法都是使用了synchronized，所以效率比ArrayList低很多

## HashMap 和 HashTable的区别

- HashMap 是线程不安全的，HashTable线程安全
- HashMap 效率高一点
- HashMap 可以有null 值，HashTable 有Null会产生NullPointerException异常
- HashTable 基本淘汰，需线程安全使用ConcurrentHashMap

## HashMap 和 ConcurrentHashMap 区别

- ConcurrentHashMap 把整个桶数组分割成很多个Segment，每个分段使用lock锁保护(1.8之后使用CAS算法)
- HashMap键值对允许有null，ConcurrentHashMap反之。

### CAS算法 compare and swap

- 	无锁算法，CAS的语义是“我认为V的值应该为A，如果是，那么将V的值更新为B，否则不修改并告诉V的值实际为多少”

## HashSet 和 HashMap 区别
![这里写图片描述](https://img-blog.csdn.net/20180911170137454?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ppd3ViYWk4ODQ5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## HashSet 检查重复

当你把对象加入HashSet时，HashSet会先计算对象的hashcode值来判断对象加入的位置，同时也会与其他加入的对象的hashcode值作比较，如果没有相符的hashcode，HashSet会假设对象没有重复出现。但是如果发现有相同hashcode值的对象，这时会调用equals（）方法来检查hashcode相等的对象是否真的相同。如果两者相同，HashSet就不会让加入操作成功。

## hashCode() 和 equals()
1.	如果两个对象相等，则hashcode一定也是相同的
2.	两个对象相等,对两个equals方法返回true
3.	两个对象有相同的hashcode值，它们也不一定是相等的
4.	综上，equals方法被覆盖过，则hashCode方法也必须被覆盖
5.	hashCode()的默认行为是对堆上的对象产生独特值。如果没有重写hashCode()，则该class的两个对象无论如何都不会相等（即使这两个对象指向相同的数据）。


[fuyuaaa](https://github.com/fuyuaaa/)