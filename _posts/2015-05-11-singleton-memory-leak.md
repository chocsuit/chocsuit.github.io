---
layout: post
title: "单例模式导致的Android内存泄露"
description: "单例模式持有activity的引用的时候有可能导致内存泄露"
category: [Android]
tags: [设计模式,内存泄露]
---

## 问题
项目开发中由于需要一个Util的单例化的存在，使用了单例模式，代码如下：

   

    public class SingletonDemo {
    
        private static volatile SingletonDemo instance;
    
        private Context mContext;
    
        private SingletonDemo(Context context) {
            this.mContext = context;
        }
    
        public static SingletonDemo getInstance(Context context) {
            if (instance == null) {
                synchronized (SingletonDemo.class) {
                    if (instance == null) {
                        instance = new SingletonDemo(context);
                    }
                }
            }
            return instance;
        }
    }

关于如何正确的创建单例模式可以参考[这篇文章](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)，在这个地方我们仅仅讨论传进context有没有问题？

## 详解

答案当然是有问题的，由于这个地方的context是个强引用，而且是被静态变量持有。在java中静态变量实在类被load的时候分配内存，在类被卸载的时候销毁。可以说static变量的生命周期伴随着进程的诞生和销毁。在Android系统中，当我们启动一个app的时候，系统启动一个进程加载一个Dalvik虚拟机实例，来负责类的加载和卸载以及GC，也就是说静态变量伴随了app进程的整个生命周期，由于上例中的instance持有了context，所以被传进来的activity（或者service或broadcast）自然没有办法得到释放，也就造成了内存泄露。

## 解决办法

其实这种需要单例的地方要仔细考虑好是否真的需要单例，因为在Android中静态变量其实是很不靠谱的，静态变量依附于进程，而Android在资源充足的时候是不会杀进程的，也就是说资源不足的情况下，进程随时可能被杀掉然后再具有资源的时候重启，那样存在静态变量里的值就很不靠谱了。如果说非得这么做不可，OK，使用context.getApplicationContext()，可以看一下Android developer上的解释：

> There is normally no need to subclass Application. In most situation, static singletons can provide the same functionality in a more modular way. If your singleton needs a global context (for example to register broadcast receivers), the function to retrieve it can be given a Context which internally uses Context.getApplicationContext() when first constructing the singleton.

Application在Android程序中是一种单例式的存在，它伴随了整个应用的始终，所以使用Application的context自然不会造成内存泄露了。

## 番外

* 在Android开发中组件之间传递参数尽量符合Android规范使用bundle来进行传递，不要使用static，因为static是依附于进程模型的，而Android中的组件如Activity，Service等是可以运行在不同的进程的。另外，由于进程的不安全，static中存储的值也自然不太可靠了。 
* 关于内存泄露的检查工具可以参考[这篇文章](http://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)的介绍。
