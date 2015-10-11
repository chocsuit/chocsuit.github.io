---
layout: post
title: "Java Weak References"
description: "对于Java中强引用，软引用，弱引用和虚引用的总结"
category: Java
tags: [java, Weak Reference]
---
{% include JB/setup %}

# 强引用
一般对Java对象的构造用的就是强引用: 

    Object o = new Object();

除非对象被置为null，否则虚拟机即使OOM也不会回收掉强引用。

# 软引用

软引用的一般用法：

    Object o = new Object();
    SoftReference<Object> softReference = new SoftReference<Object>(o);
    
只有内存不足的时候才能被GC回收掉，如果构造函数中传入了ReferenceQueue，当该引用指向的对象被标记为垃圾的时候，这个引用对象会自动地加入到引用队列里面，我们可以通过ReferenceQueue来跟踪被回收的软引用，并在适当时候将其清除掉。

# 弱引用

弱引用的一般用法：

    Object o = new Object();
    WeakReference<String> abcWeakRef = new WeakReference<String>(o);
    
当一个对象只被弱引用引用的时候就会被GC回收掉，比如将上面的代码添加一句：` o = null; `
同样的，当一个对象被标记为垃圾的时候会加入到引用队列。
下面通过一个例子来验证一下：

    public class ReferenceTest {
        private static ReferenceQueue<VeryBig> rq = new ReferenceQueue<VeryBig>();

        public static void checkQueue() {
            Reference<? extends VeryBig> ref = null;
            while ((ref = rq.poll()) != null) {
                if (ref != null) {
                    System.out.println("In queue: " + ((VeryBigWeakReference) (ref)).id);
                }
            }
        }

        public static void main(String args[]) {
            int size = 3;
            LinkedList<WeakReference<VeryBig>> weakList = new LinkedList<WeakReference<VeryBig>>();
            for (int i = 0; i < size; i++) {
                weakList.add(new VeryBigWeakReference(new VeryBig("Weak " + i), rq));
                System.out.println("Just created weak: " + weakList.getLast());

            }

            // 将第二个弱引用对象变为强引用
            VeryBig strongRef = weakList.get(1).get();

            // 检查一遍
            checkQueue();

            System.gc();
            try { // 休息几秒，让上面的垃圾回收线程运行完成
                Thread.currentThread().sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 在检查一遍
            checkQueue();

            // 检查对象
            for (int i = 0; i < size; i++) {
                if (weakList.get(i).get() == null) {
                    System.out.println("obj " + i + " is null!");
                }
            }

            // 检查是否还在引用队列
            checkQueue();
        }

        private static class VeryBig {
            public String id;
            // 占用空间,让线程进行回收
            byte[] b = new byte[2 * 1024];

            public VeryBig(String id) {
                this.id = id;
            }

            protected void finalize() {
                System.out.println("Finalizing VeryBig " + id);
            }
        }

        private static class VeryBigWeakReference extends WeakReference<VeryBig> {
            public String id;

            public VeryBigWeakReference(VeryBig big, ReferenceQueue<VeryBig> rq) {
                super(big, rq);
                this.id = big.id;
            }

            protected void finalize() {
                System.out.println("Finalizing VeryBigWeakReference " + id);
            }
        }
    }
    
运行程序：

	Just created weak: ReferenceTest$VeryBigWeakReference@7440e464
	Just created weak: ReferenceTest$VeryBigWeakReference@49476842
	Just created weak: ReferenceTest$VeryBigWeakReference@78308db1
	Finalizing VeryBig Weak 2
	Finalizing VeryBig Weak 0
	In queue: Weak 0
	In queue: Weak 2
	obj 0 is null!
	obj 2 is null!
	
OK，当调用`System.gc();`之前ReferenceQueue里面并没有数据，调用之后发现引用对象2和1的`finalize()`方法被执行，并且对象2和1被添加到了引用队列，调用引用队列的`poll()`方法即将其清除掉，最后一遍的`checkQueue()`没有发现任何对象引用。

非常重要的一点是当将weakRef.get()指向的对象被赋值给一个强引用时，在该强引用被置为null之前，都不会作为弱引用被管理。

# 虚引用

好吧，其实一直不知道虚引用有什么卵用，因为它太弱了，弱到根本无法引用。根本没有办法通过get获得它指向的对象。

其实弱引用还是有一点作用的，它的唯一作用就是当其指向的对象被回收之后，自己被加入到引用队列，用作记录该引用指向的对象已被销毁。他可以让你知道这个对象什么时候被移除掉，比如你在一个Android APP只能加载一张超大图（先不考虑图片的压缩问题），在加载下一张之前你需要确定系统已经将上一张大图片占据的内存回收掉，这时候就可以使用虚引用并结合ReferenceQueue来做判断。

# 后记

该文章主要参考了：[https://weblogs.java.net/blog/2006/05/04/understanding-weak-references](https://weblogs.java.net/blog/2006/05/04/understanding-weak-references)

代码参考了[http://blog.csdn.net/mazhimazh/article/details/19752475](http://blog.csdn.net/mazhimazh/article/details/19752475) 并做了一些改动。

关于ReferenceQueue还有很多可以说的，可以参考[http://hongjiang.info/java-referencequeue/](http://hongjiang.info/java-referencequeue/)