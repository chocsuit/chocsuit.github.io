---
layout: post
title: "Android Support Library"
description: "Android Support Library官方博客翻译"
category: [Android]
tags: [Android，Material Design]
---
（原文链接：[http://android-developers.blogspot.com/2015/05/android-design-support-library.html](http://android-developers.blogspot.com/2015/05/android-design-support-library.html)）

由于更新了整个Android用户体验的崭新设计语言material design的诞生，Android 5.0（代号：Lollipop）成为史上最重要的一个版本。从详细的规范开始学习material design 是个不错的方案，但这对于开发者，尤其是哪些考虑前向兼容的，这是个不小的挑战。有了最新的Android Design支持库的帮助，我们在其中给大家提供了许多非常重要的material design组件，并且支持Android2.1以上所有的系统版本，开发者可以使用抽屉导航组件，输入文本的浮动标签，floating action button，snackbar，tabs，并且可以使用一个支持手势和滚动的框架来将其组合使用。

###Navigation View:
对于用户尤其是首次使用某个APP的用户而言，抽屉导航极大程度上降低了在APP内进行统一的个性化和导航的难度。通过提供一个抽屉导航所需的框架和从菜单资源中构建导航条目的功能，NavigationView使得创建抽屉导航变得非常简单。

 ![enter image description here](http://3.bp.blogspot.com/-WmBBQQEJIKM/VWikAyy08sI/AAAAAAAABvc/1R36Txk83UI/s400/drawer.png)

开发者可以像使用DrawerLayout的抽屉导航布局一样使用NavigationView，如下所示：

    <android.support.v4.widget.DrawerLayout
            xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true">
    
        <!-- your content layout -->
    
        <android.support.design.widget.NavigationView
                android:layout_width="wrap_content"
                android:layout_height="match_parent"
                android:layout_gravity="start"
                app:headerLayout="@layout/drawer_header"
                app:menu="@menu/drawer"/>
    </android.support.v4.widget.DrawerLayout>
    
请注意NavigationView的两个属性：app:headerLayout用来控制头部布局，这是可选的一个属性；app:menu控制用来产生各种导航条目的菜单资源（可以在运行时更新）。NavigationView可以在大于API21的设备中和状态栏进行正确的交互。
最简单抽屉菜单可以是一系列Checkable的菜单项的集合。

    <group android:checkableBehavior="single">
        <item
            android:id="@+id/navigation_item_1"
            android:checked="true"
            android:icon="@drawable/ic_android"
            android:title="@string/navigation_item_1"/>
        <item
            android:id="@+id/navigation_item_2"
            android:icon="@drawable/ic_android"
            android:title="@string/navigation_item_2"/>
    </group>
    
为了确保用户明确知道自己选中了什么，被选中的抽屉导航的条目会高亮显示。同样可以在抽屉导航中使用subheaders来将条目分割为不同的组合。

    <item
        android:id="@+id/navigation_subheader"
        android:title="@string/navigation_subheader">
        <menu>
            <item
                android:id="@+id/navigation_sub_item_1"
                android:icon="@drawable/ic_android"
                android:title="@string/navigation_sub_item_1"/>
            <item
                android:id="@+id/navigation_sub_item_2"
                android:icon="@drawable/ic_android"
                android:title="@string/navigation_sub_item_2"/>
        </menu>
    </item>
    
通过使用setNavigationItemSelectedListener()可以为抽屉导航设置OnNavigationItemSelectedListener 监听器，当某个条目被选中的时候会产生回调。回调函数中可以得到被点击的MenuItem对象，有了该对象就可以控制选中事件，改变选中状态，加载新的内容，在程序中关闭抽屉导航，或者做其他任何你想要的操作。

###输入文本的浮动标签：(Floating labels for editing text)
在Material Design中，即便简单如EditText也得到了一定程度上的改进。在EditText中，当输入第一个字符之后，提示文本便会被隐藏掉。但是现在开发者可以使用TextInputLayout布局来包装EditText，这样当输入文本的时候，提示文本就会变成一个浮动在EditText上的标签，确保用户永远明确他们想要什么。

 ![](http://4.bp.blogspot.com/-BUKc5AwzS4A/VWihVlHr9cI/AAAAAAAABvI/rslBAoaHwzA/s320/textinputlayout.png)

通过调用setError()方法，开发者还可以在EditText的下面显示错误信息。
###Floating Action Button：
Floating Action Button是在用户界面上指示着最主要操作的圆形按钮。设计库中的FloatingActionButton通过使用主题中colorAccent着色，给开发者提供一种统一的实现。

 ![](http://2.bp.blogspot.com/-tdrgNYnQZyw/VWiOcfSRoYI/AAAAAAAABuU/6LsOxJFE4hE/s200/image03.png)

为了处理FloatingActionButton和其他组件的视觉连贯性特别严苛的情况，除了正常的大小，FloatingActionButton还支持设置最小的尺寸（fabSize=”mini”）。FloatingActionButton继承自ImageView所以开发者可以使用android:src或其他任何方法比如setImageDrawable来控制FloatingActionButton的图案。
###Snackbar：
当需要为一个操作提供轻量级的快速的反馈的时候，Snackbar是一个非常完美的选择。Snackbar以一个可选的简单地动作在屏幕或内容文字的底部出现。超过了设定的给定时长之后Snackbar自动地按照设定动画从屏幕消失。另外，在设定时长到期之前，用户也可以将Snackbar滑出屏幕。由于Snackbar具有通过滑动或其他动作和用户交互的能力，因此它比另一种轻量级的反馈机制Toast更加强大，并且开发者可以发现它们的API是相似的。

    Snackbar
      .make(parentLayout, R.string.snackbar_text, Snackbar.LENGTH_LONG)
      .setAction(R.string.snackbar_action, myOnClickListener)
      .show(); // Don’t forget to show!

应该注意的是make()方法的第一个参数是View的一个对象，Snackbar会尝试寻找一个以其底部作为锚点的合适的父类布局。
###Tabs：
在App中通过Tabs来切换不同的View并不是material design新提出的概念，但是它却是在APP中用来组织不同分类信息的最高等级的导航模式（比如，音乐的不同体裁）。
 
![enter image description here](http://1.bp.blogspot.com/-liraQhLEn60/VWihbiaXaJI/AAAAAAAABvQ/nKi1_xcx6yk/s320/tabs.png)

设计库中的TabLayout既实现了View宽度等距离分割的fix tabs，也实现了大小不一，可以水平滑动的scrollable tabs。Tabs可以在代码中添加：

    TabLayout tabLayout = ...;
    tabLayout.addTab(tabLayout.newTab().setText("Tab 1"));

然而，如果开发者使用ViewPager进行Tab之间的水平分页，可以直接通过PagerAdapter的getPageTitle()方法然后调用setupWithViewPager()方法将二者联系起来，从而直接创建Tab。这保证了Tab被选中时更新ViewPager并且ViewPager切换的时候更新Tab。
###CoordinatorLayout，手势和滚动：
特殊的视觉效果只是material design一个方面，手势操作也是一个符合material design应用非常重要的一部分。Material Design在包括了如触摸波浪效果和有意义的转换动画的同时，还引入了CoordinatorLayout，这个布局可以为字类视图之间的触摸事件提供另一种层面上的控制，再设计库中的很多组件都进行了利用。
###CoordinatorLayout和floating action button:
这两种控件有一个非常棒的使用方式，将floating action button作为CoordinatorLayout的子元素，并且将CoordinatorLayout传递给Snackbar的make()方法，FloatingActionButton利用CoordinatorLayout提供的额外的回调，可以在Snackbar显示的时候自动上移，并在Snackbar消失的时候自动回归原来的位置，这样，Snackbar就不是浮在floating action button上面了，这在所有Android3.0及以上的设备中都可以实现，不需要多余的代码。
在CoordinatorLayout中，layout_anchor属性和layout_anchorGravity属性配合，可以用来设定浮动视图（比如floating action button）之间的相对位置。
###CoordinatorLayout和app bar:
另一个关于CoordinatorLayout非常主要的用法就是与app bar（之前的action bar）和滚动策略的配合使用。Toolbar可以更加简单地定制这个APP中标志性的部分的外观和交互，而许多开发者可能已经开始使用它了。而设计库将这个提升了一个等级：通过AppBarLayout，Toolbar和其他视图（比如提供Tab的TabLayout）可以在标记了ScrollingViewBehavior的兄弟视图中和滚动事件交互。因此，开发者可以像这样创建一个布局：

     <android.support.design.widget.CoordinatorLayout
            xmlns:android="http://schemas.android.com/apk/res/android"
            xmlns:app="http://schemas.android.com/apk/res-auto"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
         
         <! -- Your Scrollable View -->
        <android.support.v7.widget.RecyclerView
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                app:layout_behavior="@string/appbar_scrolling_view_behavior" />
    
        <android.support.design.widget.AppBarLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content">
       <android.support.v7.widget.Toolbar
                      ...
                      app:layout_scrollFlags="scroll|enterAlways">
    
            <android.support.design.widget.TabLayout
                      ...
                      app:layout_scrollFlags="scroll|enterAlways">
         </android.support.design.widget.AppBarLayout>
    </android.support.design.widget.CoordinatorLayout>

这样，当用户滚动RecyclerView的时候AppBarLayout可以根据子类视图的滚动标志来控制他们进入屏幕和离开屏幕的方式。这些滚动标志包括：
•	scroll：所有想要滑出屏幕的视图都应该设置这个标志，对于那些未设置的视图，会固定在屏幕顶端。
•	enterAlways：这个标志确保了向下滚动的时候视图可见，使quick return模式可用。
•	enterAlwaysCollapsed：当一个视图被设置了minHeight并且使用了这个标志之后，那么这个视图只会按照最小的高度进入屏幕，当可滚动的视图到达顶部的时候，它会展开完整的高度。
•	exitUntilCollapsed：设置了这个标志之后，视图在离开屏幕之前会先滚动到折叠状态（最小高度）。

`注意：所有使用scroll这个标志的视图都应该在不使用这个标志的视图之前声明，保证所有的视图从顶部离开屏幕，将固定不动的视图留在下面。`
###Collapsing Toolbar：
如果直接在AppTabLayout中添加Toolbar可以通过设置enterAlwaysCollapsed和exitUntilCollapsed滚动标志来控制折叠效果，但是这并没有对不同元素交互时的细节控制能力。因此开发者可以使用CollapsingToolbarLayout：

    <android.support.design.widget.AppBarLayout
            android:layout_height="192dp"
            android:layout_width="match_parent">
        <android.support.design.widget.CollapsingToolbarLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                app:layout_scrollFlags="scroll|exitUntilCollapsed">
            <android.support.v7.widget.Toolbar
                    android:layout_height="?attr/actionBarSize"
                    android:layout_width="match_parent"
                    app:layout_collapseMode="pin"/>
            </android.support.design.widget.CollapsingToolbarLayout>
    </android.support.design.widget.AppBarLayout>

通过对CollapsingToolbarLayout设置app:layout_collapseMode=”pin”可以保证当视图被折叠的时候Toolbar固定在屏幕顶端。更棒的是，当CollspsingToolBarLayout和Toolbar一起使用的时候，当布局完全可见的时候标题会变大，并且在布局折叠的时候转变到默认大小。需要注意的是，应该为CollapsingToolbarLayout调用setTitle()方法设置标题，而不是对Toolbar设置。
除了固定视图以外，还可以使用app:layoutCollapseMode=”parallax”（还可以通过app:layout_collapseParallaxMultiplier="0.7"来设置视差率），来实现视差滚动（比如说在CollapsingToolbarLayout中添加ImageView作为兄弟视图）。这种情景配合CollapsingToolbarLayout 的app:contentScrim="?attr/colorPrimary" 属性会在视图折叠的时候产生非常棒的效果。
###CoordinatorLayout 和个性化视图
需要注意的非常重要的一点就是CoordinatorLayout并不知道FloatingActionButton或者AppBarLayout的作用，它仅仅以Coordinator.Behavior的形式提供一个额外的API接口，来更好的控制子类View之间的触摸事件和手势，并且通过onDependentViewChanged()来声明依赖和接受回调。
视图可以通过在代码中设置CoordinatorLayout.DefaultBehavior(YourView.Behavior.class) 或者在布局文件中设置app:layout_behavior="com.example.app.YourView$Behavior"属性来控制默认的表现样式。Android从框架级别保证了任意视图和CoordinatorLayout交互的可能性。
###Available Now：
设计库现在就可以使用了，只要保证在SDK管理器中更新了Android 支持库即可。只要添加下面一行依赖：

     compile 'com.android.support:design:22.2.0'
    
需要注意的是，设计库依赖了support v4和AppCompat支持库，当使用设计库的时候会自动包含二者。这些新的组件在Android Studio的布局编辑器的设计视图中是可见的，这给了开发者一个更加方便的预览这些新的组件的方式。
设计库，AppCompat和其他Android支持库都是让开发者避免从零开始构建一个现代化、外观精美的Android应用非常有用的工具。
