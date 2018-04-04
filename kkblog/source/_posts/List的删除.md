---
title: List遍历要注意的坑
date: 2017-09-04 11:23:11
tags: [Java,list,遍历]
categories: Java
---
List的删除测试：

主要函数：

```java
public static void main (String[] args){
	List<String> myList = new ArrayList<String>();
	myList.add("a");
	myList.add("b");
	myList.add("c");
	myList.add("d");
	myList.add("e");
	test1(myList,"d");
}
```

```java
public static void show(List<String> list){
		for(String e : list){
			System.out.println(e);
		}
	}
```

### **首先是根据下标遍历：**

#### Test1：

很明显，事先的size在remove发生后会改变，所以出现了溢出。

```java
// size遍历，删除指定下标。TEST1
	public static void test1(List<String> list, String key){
		int size = list.size();
		for(int i = 0; i < size; ++i){
			String tmp = list.get(i);
			if(tmp.equals(key)){
				list.remove(key);// 用list.remove(i)也是一样结果
				System.out.println("After removing, length is " + list.size());
			}
		}
		show(list);
	}
```

![result test1](http://ovwunej09.bkt.clouddn.com/listR01.png)

#### TEST2：

当每个元素都不同时，可以正确删除，一开始看还以为是对的...但是当出现重复元素时，只能删除其中一个元素。


```java
public static void main (String[] args){
	List<String> myList = new ArrayList<String>();
	myList.add("a");
	myList.add("b");
	myList.add("c");
	myList.add("c");
	myList.add("d");
	myList.add("e");
	test2(myList,"c");
}
```
```java
// size不固定，和i比较时再取.TEST2
	public static void test2(List<String> list, String key){
		for(int i = 0; i < list.size(); ++i){
			String tmp = list.get(i);
			if(tmp.equals(key)){
				list.remove(key);// 用list.remove(i)也是一样结果
				System.out.println("After removing, length is " + list.size());
			}
		}
		show(list);
	}
```
![clipboard.png](http://ovwunej09.bkt.clouddn.com/listR03.png)


因为执行remove时，将被删除的元素之后的元素全部往前移动一位，那么原来的size-1位置就为null了，最后把size--就可以了。比如i==2出现了删除，i==3的元素和2一样，但是它去了2位置，这时就扫描不到它了。

#### TEST3：

那么可能会想到，既然是被删除之后的元素往前面移动，那么我们可以试试从后遍历\~！这样后面的元素往前移动也没有关系，因为他们已经被扫描过了。

```java
// size不固定，和i比较时再取.从后往前遍历.TEST3
	public static void test3(List<String> list, String key){
		for(int i = list.size()-1; i >=0 ; --i){
			String tmp = list.get(i);
			if(tmp.equals(key)){
				list.remove(key);// 用list.remove(i)也是一样结果
				System.out.println("After removing, length is " + list.size());
			}
		}
		show(list);
	}
```

![result test3](http://ovwunej09.bkt.clouddn.com/listR04.png)

成功了\~！！！
先固定size的方法，一旦发生remove肯定会导致size改变，肯定行不通，不再测试。

### **for each遍历**

#### TEST4：

for each遍历，有相同就直接删除：

```java
// for each.遍历CowList，删除list。TEST4
		public static void test4(List<String> list, String key){
			int count = 0;
			for(String tmp:list){
				if(tmp.equals(key)){
					System.out.println(count + " is current index");
					list.remove(key);// 用list.remove(i)也是一样结果
					System.out.println("After removing, length is " + list.size());
				}
				count++;
			}
			show(list);
		}
```

![clipboard.png](http://ovwunej09.bkt.clouddn.com/listR05.png)

这个错误，研究过modCount就能明白，迭代器发生了错误。

在remove的时候modCount--，但是迭代器的expectedModcount是最开始遍历前的modcount，modCount值不是期望值，就抛出了异常。

这个fail-fast迭代是为了保证线程安全的，put这些操作也会用到。

#### TEST5：

那么foreach怎么修改才能成功删除呢...最开始想的时候没想出来，有的人说remove后就break也是厉害了....不管重复元素了吗?


看到一种方法是把List复制为COWArrayList，写时复制，也就是remove完才把结果存入COWlist...但这样实际上操作的是COWAraayList而不是原本的List，感觉也不太可取。

**但是，假设遍历COWArrayList，但是操作原本的List呢？？**

```java
// for each.遍历CowList，删除list。TEST5
	public static void test5(List<String> list, String key){
		int count = 0;
		CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<String>(list);
		for(String tmp:cowList){
			if(tmp.equals(key)){
				System.out.println(count + " is current index");
				list.remove(key);// 用list.remove(i)也是一样结果
				System.out.println("After removing, length is " + list.size());
			}
			count++;
		}
		show(list);
	}
```

**我有点惊呆...居然成功了- -**

![clipboard.png](http://ovwunej09.bkt.clouddn.com/listR06.png)
