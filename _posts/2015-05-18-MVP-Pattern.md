---
layout: post
title: "Android开发中的MVVM模式实践"
description: "MVVM(Model,View,ViewModel)实现表现层解耦的实践"
category: [Android]
tags: [MVVM,解耦]
---

# MVC, MVP and MVM

一个典型的MVC模式如下图所示：
![MVC](https://github.com/techsummary/techsummary.github.io/blob/master/assets/image/MVC.png)

MVC模式在实践中其实并不适合大规模Android应用的开发，因为Android从的设计上来说，activity/fragment更像是View层的组件和controller的结合，因为单纯的XML布局不能完成非常复杂的交互逻辑。工作项目由于一开始（2012年左右）的设计是MVC的，activity承担了view和controller的角色，工作中一个Activity耦合了展现层的逻辑和controller的逻辑后达到了4K+的行数，一个非常恐怖的数字。所以必须要对之进行解耦重构，其中MVP和MVVM引起了我们的注意，MVP模式见下图：

![MVP](https://github.com/techsummary/techsummary.github.io/blob/master/assets/image/MVP.jpg)

MVP模式是从MVC派生出来的，但是它阻止了model和view的交流，使业务逻辑与展示解耦，MVP模式中，activity承担的是View的角色，Presenter和VIew互相依赖（为了解耦有的时候会使用接口和抽象类，这个其实有点MVVM的意思了），Presenter是View和Model的一个桥梁。Presenter和View是一一对应的，是View的一种抽象。MVP模式在解耦的层面做的不如MVVM彻底。关于MVP模式的讨论可以参见[这篇文章](http://antonioleiva.com/mvp-android/).

MVVM模式是在MVP模式上进一步解耦而派生出来的，见下图：

![MVVM](https://github.com/techsummary/techsummary.github.io/blob/master/assets/image/MVVM.png)

MVVM和MVP模式的区别在于View对于ViewModel来说是透明的，ViewModel对于Model来说是透明的，ViewModel持有View的回调，Model持有ViewModel的回调，这样就实现View，ViewModel，Model的解耦。

更多关于MVC, MVP, MVVM模式的讨论可以看一下[这篇文章](http://blogs.k10world.com/technology/difference-between-mvc-vs-mvp-vs-mvvm/).

# Demo

以一个点击button发送网络请求，并对结果进行处理的demo为例。
`Activity`的代码如下：

    public class MVVMDemoActivity extends ActionBarActivity
        implements IViewModelCallback {
    
        MVVMDemoViewModel viewModel;
    
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_mvvmdemo);
    
            viewModel = new MVVMDemoViewModel(this);
            findViewById(R.id.sendrequest).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    viewModel.action();
                }
            });
        }
    
        @Override
        public void onDataReceive() {
            //数据回来后进行视图的更新
        }
    
        @Override
        public void onError() {
            //数据请求失败的时候做一些处理
        }
    }

可以看到Activity中只负责了View层的展示逻辑，毫无接收参数设置参数等不属于View的操作。

`ViewModel`的代码如下：

    public class MVVMDemoViewModel implements IRequestCallback {
    
        IViewModelCallback viewListener;
        DataRequestModel model;
    
        public MVVMDemoViewModel(IViewModelCallback viewListener) {
            this.viewListener = viewListener;
            model = new DataRequestModel(this);
        }
    
        public void action() {
            //这里可以做一些参数的初始化操作
            //......
            model.sendRequest();
        }
    
        @Override
        public void onSuccess() {
            viewListener.onDataReceive();
        }
    
        @Override
        public void onError() {
            viewListener.onError();
        }
    }

View层对于ViewModel来说是透明的，ViewModel只负责对实现了IViewModelCallback类型进行操作。ViewModel可以做一些参数的获取和设置等既不属于View层也不是Model层面的操作，可以实现视图展示层和业务代码的解耦。

`Model`层的代码如下：

    public class DataRequestModel {
        private IRequestCallback IRequestCallback;
    
        public DataRequestModel(IRequestCallback IRequestCallback) {
            this.IRequestCallback = IRequestCallback;
        }
    
        public void sendRequest() {
            //***模拟网络请求
            //connecting...
            //网络请求成功
            IRequestCallback.onSuccess();
            //网络请求失败
            IRequestCallback.onError();
        }
    }

View和ViewModel对于Model来说都是透明的，Model只负责进行网络请求，这个业务是非常独立的，可以说放之四海而皆准。

两个接口的代码如下：

    public interface IViewModelCallback {
        void onDataReceive();
        void onError();
    }
    
    public interface IRequestCallback {
        void onSuccess();
        void onError();
    }


# 总结
可见，MVVM在比较大规模的Android应用的开发中可以做到一个很好的解耦，工作实践中也证实了这一点。