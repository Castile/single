# java设计模式-单例设计模式（一）
#### Time: 12/4/2017 9:17:18 PM ####
#### Author: Hongliang Zhu ####
#### Email:hongliangzhu1002@163.com
---

## **单例模式第一版：懒汉模式**
 
    public class Singleton {
    
    	private Singleton() {}  //私有构造函数
    
   		private static Singleton instance = null; //单例对象
    
       //静态工厂方法
    
    	public static Singleton getInstance() {
    
  
   			if (instance == null) {
    
    		instance = new Singleton();
    
   		 }
    
    	return instance;
    
    	}
    }
    
### 关键要点： ###
1. 要想让一个类只能构建一个对象，自然不能让它随便去做new操作，因此*signleton*的构造方法是私有的。

2. instance是singleton类的静态成员，也是我们的单例对象。它的初始值可以写成null，也可以写成new singleton()。至于其中的区别后来会做解释。

3. **getinstance**是获取单例对象的方法。

> 如果单例初始值是null，还未构建，则构建单例对象并返回。这个写法属于单例模式当中的**懒汉模式**。
> 如果单例对象一开始就被new singleton()主动构建，则不再需要判空操作，这种写法属于**饿汉模式**。
> 这两个名字很形象：**饿汉主动找食物吃，懒汉躺在地上等着人喂。**

----------
### 但是以上的这段代码并非线程安全的，为什么说刚才的代码不是线程安全呢？： ###
	假设Singleton类刚刚被初始化，instance对象还是空，这时候两个线程同时访问getInstance方法：


![](https://i.imgur.com/Z01RSln.png)

	因为Instance是空，所以两个线程同时通过了条件判断，开始执行new操作：


![](https://i.imgur.com/1t9iPZV.png)

	这样一来，显然instance被构建了两次。让我们对代码做一下修改：

## 单例模式第二版：线程安全 ##


   	 public class Singleton {
    
    	private Singleton() {}  //私有构造函数
    
      	private static Singleton instance = null;  //单例对象
    
      	 //静态工厂方法
    
        public static Singleton getInstance() {
    
    	if (instance == null) {  //双重检测机制
    
     		synchronized (Singleton.class){  //同步锁
    
       		if (instance == null) { //双重检测机制
    
     			instance = new Singleton();
    
       			}
    
    		}
    
    	 }
    
    		return instance;
    
    	}
    
 	}


###  为什么这样写呢？我们来解释几个关键点：  ###
1. 为了防止new Singleton被执行多次，因此在new操作之前加上Synchronized 同步锁，锁住整个类（注意，这里不能使用对象锁）。
2. 进入Synchronized 临界区以后，还要再做一次判空。因为当两个线程同时访问的时候，线程A构建完对象，线程B也已经通过了最初的判空验证，不做第二次判空的话，线程B还是会再次构建instance对象。

![](https://i.imgur.com/kL0p2VD.png)
--------
![](https://i.imgur.com/BrwBc8y.png)
--------
![](https://i.imgur.com/0O05gVS.png)
-------
![](https://i.imgur.com/yl1EHgG.png)
------
![](https://i.imgur.com/watSTLt.png)
----

> ## 像这样两次判空的机制叫做双重检测机制 ##

 
> 	但是，上面的单例模式第二版并非绝对的线程安全：
> 	假设这样的场景，当两个线程一先一后访问getInstance方法的时候，当A线程正在构建对象，B线程刚刚进入方法：

![](https://i.imgur.com/bsr527l.png)
---

这种情况表面看似没什么问题，要么Instance还没被线程A构建，线程B执行 if（instance == null）的时候得到false；要么Instance已经被线程A构建完成，线程B执行 if（instance == null）的时候得到true。

真的如此吗？答案是否定的。这里涉及到了JVM编译器的指令重排。

指令重排是什么意思呢？比如java中简单的一句 instance = new Singleton，会被编译器编译成如下JVM指令：

memory =allocate();    //1：分配对象的内存空间 

ctorInstance(memory);  //2：初始化对象 

instance =memory;     //3：设置instance指向刚分配的内存地址 

但是这些指令顺序并非一成不变，有可能会经过JVM和CPU的优化，指令重排成下面的顺序：

memory =allocate();    //1：分配对象的内存空间 

instance =memory;     //3：设置instance指向刚分配的内存地址 

ctorInstance(memory);  //2：初始化对象 

当线程A执行完1,3,时，instance对象还未完成初始化，但已经不再指向null。此时如果线程B抢占到CPU资源，执行  if（instance == null）的结果会是false，从而返回一个没有初始化完成的instance对象。如下图所示：
![](https://i.imgur.com/COXoiFS.png)
-----

![](https://i.imgur.com/9MuByIV.png)
---
---
> 如何避免这一情况呢？我们需要在instance对象前面增加一个修饰符volatile。

----------

## 单例模式第三版：修饰符volatile ##

    public class Singleton {
    
    	private Singleton() {}  //私有构造函数
    
       	private volatile static Singleton instance = null;  //单例对象
    
       	//静态工厂方法
    
       	public static Singleton getInstance() {
    
       		if (instance == null) {  //双重检测机制
    
       			synchronized (this){  //同步锁
    
     				if (instance == null) { //双重检测机制
    
       					instance = new Singleton();
    
     				}
    
      			}
    
       		}
    
       		return instance;
    
    	}
    
    }

 
那么问题来了，volatile到底是个什么玩意儿呢？在维基百科中是这样说的:
> The volatile keyword indicates that a value may change between different accesses, it prevents an optimizing compiler from optimizing away subsequent reads or writes and thus incorrectly reusing a stale value or omitting writes. 

###  简单点说就是，volatile修饰符阻止了变量访问前后的指令重排，保证了指令的执行顺序。  ###
经过volatile的修饰，当线程A执行instance = new Singleton的时候，JVM执行顺序是什么样？始终保证是下面的顺序：

1. memory =allocate();    //1：分配对象的内存空间 

2. ctorInstance(memory);  //2：初始化对象 

3. instance =memory;     //3：设置instance指向刚分配的内存地址 

	如此在线程B看来，instance对象的引用要么指向null，要么指向一个初始化完毕的Instance，而不会出现某个中间态，保证了安全。

---------
## 说了那么多，其实这种方法还是有漏洞的，如果通过 【反射】 的方式，仍然可以构建多个对象。这个问题我会在单例设计模式（二）中解决，在哪里我会介绍一种更为优雅的单例设计模式实现  ##





# 说明： #
- volatile关键字不但可以防止指令重排，也可以保证线程访问的变量值是主内存中的最新值。

----



12/4/2017 9:45:34 PM   工大软件园
