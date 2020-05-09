* 本文将从以下几个地方来进行介绍
  1. 内部结构
  2. 场景
  3. 问题-回答
  4. 副作用
- 内部结构
  - 说下几个类的关系
    - ThreadLocalMap 是 ThreadLocal 的内部类(真正的数据就是向这里面放的).
    - Thread 内部有个 ThreadLocalMap 的属性, 叫 threadLocals. 真正存放的数据, 是向这个里面放的.
    - 说白了就是: Thread内部, 有这么行代码: ThreaddLocal.ThreadLocalMap threadLocals = null;//初始化为null.
  - 当我们放数据时, 通过ThreadLocal, 放到了 Thread 的 ThreadLocalMap 中(数据其实还是在Thread内). 当线程调用另一个ThreadLocal对象的set()方法时, 其实就是在线程的ThreadLocalMap对象中, 多了一个Entry而已.

看看ThreadLocal的get、set、remove方法.

====================get()======================================

```java
private T get(){
    //获取当前线程
    Thread t = Thread.currentThread();
    //找到线程内部的ThreadLocalMap对象: return t.threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //从线程的ThreadLocalMap对象内部找到此ThreadLocal对应的Value.
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    //完成初始化, 并设置值.
    return setInitialValue();
}

private T setInitialValue() {
    //默认返回null: return null;
    T value = initialValue();
    Thread t = Thread.currentThread();
    //找到当前线程的ThreadLocalMap, 不为null就设置值, 为null就创建出来, 并将初始化一下(将当前的ThreadLocal和默认的Valueput进去).
    ThreadLocalMap map = getMap(t);
    if (map != null){
        map.set(this, value);
    }else {
        //t.threadLocals = new ThreadLocalMap(this, firstValue);
        createMap(t, value);
    }
    return value;
}
```

=======================set()======================================

//其实就是向当前线程的ThreadLocalMap中放数据.

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

=====================remove()=========================

//从当前线程的ThreadLocalMap中移除数据 

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if(m != null) {
        m.remove(this);
    }
}
```

ThreadLocal的get()、set()、remove()会调用清理函数, 但get()并不是百分百会调用, 

从上面的代码可以知道, 此层调用用的是 ThreadLocalMap的getEntry()、set()、remove()方法. 下面会介绍的.

ThreadLocalMap的结构: 

```java
//初始值为16.
private static final int INITIAL_CAPACITY = 16;
//存数据.
private Entry[] table;
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    //构造器.
    Entry(ThreadLocal<?> k, Object v) {
    	super(k);
    	value = v;
	}
}
```

//查找下一个位置: 当越界了之后, 会从0位置开始找. 所以说, 是个环形数组. 下面的 prevIndex也一样.

```java
private static int nextIndex(int i, int len) {
	return ((i + 1 < len) ? i + 1 : 0);
}
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

========================getEntry()===============================

总结

哈希值确定数组下标, 到指定槽位去找, 找不到的话, 就向后查找(开放地址法)开始遍历, 直到找到对应的数据为止. 

而且查找的过程中如果发现K为null的槽位, 会从那个槽位开始做一次连续段的清理工作, 也就是 expungeStaleEntry()函数. 连续段清理指的是, 如果发现槽位的Entry是null了, 它就会停止清理工作.

```java
private Entry getEntry(ThreadLocal<?> key) {
	// 通过hashcode确定下标
	int i = key.threadLocalHashCode & (table.length - 1);
	Entry e = table[i];
    // 如果找到则直接返回 
	if (e != null && e.get() == key) {
 		return e;
 	} else{
 	// 找不到的话接着从i位置开始向后遍历，基于线性探测法，是有可能在i之后的位置找到的
 		return getEntryAfterMiss(key, i, e);
	}
}
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
	Entry[] tab = table;
	int len = tab.length;
	// 循环向后遍历
	while (e != null) {
 		// 获取节点对应的k
 		ThreadLocal<?> k = e.get();
 		// 相等则返回
 		if (k == key){
 			return e;
 		}
 		// 如果为null，触发一次连续段清理
 		if (k == null){
 			expungeStaleEntry(i);
 		}else{
 			// 获取下一个下标接着进行判断
 			i = nextIndex(i, len);
 		}
 		e = tab[i];
 	}
 	return null;
}
```

============核心清理函数

总结: 从指定位置开始, 向后遍历, 对途中K为null的数据, 进行清理工作(将Value设为null), size-1. 对K不为null的数据进行rehash, rehash出来的槽位, 如果不是当前位置的话, 会将这个结点迁移到指定位置后面的第一个为null的位置. 

```java
private int expungeStaleEntry(int staleSlot) {
	// 新的引用指向table
 	Entry[] tab = table;
 	// 获取长度
 	int len = tab.length;
 	// 先将传过来的下标置null
 	tab[staleSlot].value = null;
 	tab[staleSlot] = null;
 	// table的size-1
 	size--;
 	Entry e;
 	int i;
 	// 遍历删除指定节点所有后续节点当中，ThreadLocal被回收的节点
 	for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        // 获取entry当中的key
        ThreadLocal<?> k = e.get();
        // 如果ThreadLocal为null，则将value以及数组下标所在位置设置null，方便GC
        // 并且size-1
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else { // 如果不为null
            // 重新计算key的下标
            int h = k.threadLocalHashCode & (len - 1);
            // 如果是当前位置则遍历下一个
            // 不是当前位置，则重新从i开始找到下一个为null的坐标进行赋值
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null){
                    h = nextIndex(h, len);
                }
                tab[h] = e;
            }
        }
 	}
	return i;
}
```

======================================set()=============================================

总结: 找到对应槽位, K不同的话, 线性探测, 向后查找. K相同的话, 直接替换Value. K为null的有值的话, 替换掉, 并且调用上面说的那个清理函数.

 replaceStaleEntry()、cleanSomeSlots() 都会调用上面的清理函数.

```java
private void set(ThreadLocal<?> key, Object value) {
 	// 新开一个引用指向table
 	Entry[] tab = table;
 	// 获取table的长度
 	int len = tab.length;
 	// 获取对应ThreadLocal在table当中的下标
 	int i = key.threadLocalHashCode & (len-1);
 	for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
 		ThreadLocal<?> k = e.get();
 		if (k == key) {
 			e.value = value;
 			return;
 		}
 		// 如果 k 为null，则替换当前失效的k所在Entry节点
 		if (k == null) {
 			replaceStaleEntry(key, value, i);
 			return;
 		}
 	}
 	// 找到空的位置，创建Entry对象并插入
 	tab[i] = new Entry(key, value);
 	// table内元素size自增
 	int sz = ++size;
 	if (!cleanSomeSlots(i, sz) && sz >= threshold){
 		rehash();
 	}
}
```

========================remove()========================================

总结: 找到槽位, K不同的话, 线性探测, 找到的话, 清理掉. 然后调用清理函数, 

```java
// 将ThreadLocal对象对应的Entry节点从table当中删除
private void remove(ThreadLocal<?> key) {
 	Entry[] tab = table;
 	int len = tab.length;
 	int i = key.threadLocalHashCode & (len-1);
 	for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
 		if (e.get() == key) {
 			// 将引用设置null，方便GC
 			e.clear();
 			// 从该位置开始进行一次连续段清理
 			expungeStaleEntry(i);
 			return;
 		}
 	}
}
```

![Image](resources\Image.png)

原理图-1(源自《码出高效》)

![Image2](resources\Image2.png)

原理图-2

- 场景
  - 局部变量 在 方法内 各代码块 间进行传递
  - 类内变量 在 类的方法间进行传递。
  - 复杂的线程方法 需要调用很多方法来实现某个功能 这时候可以使用 ThreadLocal 来传递线程内变量
  - 它通常用于同一个线程内，跨类、跨方法传递数据。
  - 如果没有 ThreadLocal, 那么相互之间的信息传递，需要要靠返回值和参数, 那么 类 或者 框架之间 会 互相耦合。
- 问题-回答
  - 为什么一般设置为静态的呢?
    - 设置静态的, 是为了让他可以在很多地方, 都能访问到. 假如 A、B 两个方法在不同的类中, 需要依赖一个ThreadLocal对象传递数据, 那怎么做才能让 不同类的不同方法都访问到同一个对象呢? 设置为静态的比较常用而已.
  - 为什么要将K设置为弱引用
    - JDK的原意, 是在ThreadLocal对象不想再使用的时候(比如说你把它置成了null, 希望被回收掉), 那么线程再持有这个ThreadLocal的引用是没有意义的, 应该将它回收掉, 防止 "内存泄漏". 设置成弱引用, 下一轮就直接回收掉了.
  - ThreadLocal为什么会存在内存泄漏
    - ThreadLocal 在没有外部对象强引用时，发生GC, 弱引用的Key会被回收(ThreadLocal为K, 放在了Entry里面了嘛)，
    - 而Value不会回收，线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露.
  - Value不回收, 造成内存泄漏, 那为什么它不设置为弱引用呢?
    - 假如说, 我们的Value, 没有外部的引用了, 那么此时就只有Entry内部的一个弱引用. 那么GC来了, Value就直接被干掉了, 我线程想从里面拿数据, 然后你跟我说没了, 被干掉了. 那这怎么整啊?
    - Value不像K, K在外部起码还有一个强引用呢(因为ThreadLocal一般设置为静态的嘛, 肯定不会被回收掉), Value就不行了, GC一来, 直接没了.
- 副作用
  - 内存泄漏
    - 不多说了,
  - 脏数据: 来自《码出高效》
    - 线程复用 用会产生脏数据。由于结程池会重用 Thread 对象 ，那么与 Thread 绑定 的 ThreadLocal 变量也会被重用。如果在实现的线程 run()方法体 中不显 式地调用 remove() 清理与线程相关的 TbreadLocal 信息，那么倘若下一个结程不调用 set() 设置初始值，就可能 get() 到之前线程设置的对象信息.
  - 下面两个, 也不能说算是问题吧
    - ThreadLocalMap对冲突的处理是进行线性探测, 所以在get()时, 可能会遍历整个数组. 而且遍历过程中还可能会进行清理工作.
    - 在清理时, 从当前位置, 向后对槽位进行遍历, 当前元素是为null的话, 也就直接退出了. 所以说, 可能会有部分槽位清理不到.