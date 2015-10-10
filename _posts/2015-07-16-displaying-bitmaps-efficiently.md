---
layout: post
title: "Displaying Bitmaps Efficiently"
description: "Google官方教程学习笔记"
category: Android
tags: [Android, Bitmap, Memory]
---

*Google官方教程原文链接：[http://developer.android.com/training/displaying-bitmaps/index.html](http://developer.android.com/training/displaying-bitmaps/index.html)*

# 1. 高效加载大图

## 1.1 读取bitmap的dimension和type
BitmapFactory提供了一些方法来通过不同的途径创建bitmap，比如：decodeByteArray(), decodeFile(), decodeResource()等。但是这些方法都试图为图片分配内存，因此很容易OOM。为此每种方法都有一个BitmapFactory.Options参数，为该参数设置inJustDecodeBounds为true就可以在对图片解码的时候不分配内存。bitmap会返回null，但是bitmap的outWidth，outHeight，和outMimeType却可以获得。

    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
    int imageHeight = options.outHeight;
    int imageWidth = options.outWidth;
    String imageType = options.outMimeType;

为了避免OOM，记得对图片解码之前检查一下dimension。
##1.2 将图片质量降级加载进内存
将一张1024*768的大图加载进一个128*96尺寸的ImageView是非常不值得的，为此我们需要将图片进行降级操作。在bitmap的configuration为ARGB_8888的情况下，吧BitmapFactory.Options的inSampleSize属性设置为4，就可以把一张2048*1536的图片降级为 512*384的，内存占用从12M降低为0.75M。

	public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth,     int reqHeight) {
	    // Raw height and width of image
	    final int height = options.outHeight;
	    final int width = options.outWidth;
	    int inSampleSize = 1;
	
	    if (height > reqHeight || width > reqWidth) {
	
	        final int halfHeight = height / 2;
	        final int halfWidth = width / 2;
	
	        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
	        // height and width larger than the requested height and width.
	        while ((halfHeight / inSampleSize) > reqHeight
	                && (halfWidth / inSampleSize) > reqWidth) {
	            inSampleSize *= 2;
	        }
	    }
	
	    return inSampleSize;
	}

总体的降级加载大图的流程如下所示：

	public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
	        int reqWidth, int reqHeight) {
	
	    // First decode with inJustDecodeBounds=true to check dimensions
	    final BitmapFactory.Options options = new BitmapFactory.Options();
	    options.inJustDecodeBounds = true;
	    BitmapFactory.decodeResource(res, resId, options);
	
	    // Calculate inSampleSize
	    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
	
	    // Decode bitmap with inSampleSize set
	    options.inJustDecodeBounds = false;
	    return BitmapFactory.decodeResource(res, resId, options);
	}

通过下面这行代码可以轻松地将一个大图变为一个100*100的小图。

    mImageView.setImageBitmap(
        decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));

# 2. 避免在UI线程处理Bitmap
一句话：只要不是从内存中读取图片并处理，都应该放到后台线程中，以免阻塞UI线程。
## 2.1 使用AsyncTask
不考虑并发问题的话可以使用AsyncTask这么处理：

    class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
        private final WeakReference<ImageView> imageViewReference;
        private int data = 0;
    
        public BitmapWorkerTask(ImageView imageView) {
            // Use a WeakReference to ensure the ImageView can be garbage collected
            imageViewReference = new WeakReference<ImageView>(imageView);
        }
    
        // Decode image in background.
        @Override
        protected Bitmap doInBackground(Integer... params) {
            data = params[0];
            return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
        }
    
        // Once complete, see if ImageView is still around and set bitmap.
        @Override
        protected void onPostExecute(Bitmap bitmap) {
            if (imageViewReference != null && bitmap != null) {
                final ImageView imageView = imageViewReference.get();
                if (imageView != null) {
                    imageView.setImageBitmap(bitmap);
                }
            }
        }
    }

使用弱引用可以很好地避免ImageVIew的内存泄露，当然，在任务执行完成的时候ImageView不能保证还在，因此需要检测引用是否为null。
##2.2 处理并发
像在ListView或GridView中，一般会启动非常多的后台线程来加载图片（从网络或者磁盘或者其他来源），这样就不能保证当线程任务执行完成的时候，响应的View没有被回收掉，更加不可能保证任务结束的顺序和开始的顺序一致。
这篇文章[ Multithreading for Performance ](http://android-developers.blogspot.com/2010/07/multithreading-for-performance.html)讨论了一种并发问题的解决方案。让ImageView存储一个最近启动的AsyncTask的引用，当任务完成可以检测到它。创建一个Drawable的子类来存储工作任务的引用。在这里继承BitmapDrawable来显示一张占位图。

    static class AsyncDrawable extends BitmapDrawable {
        private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;
    
        public AsyncDrawable(Resources res, Bitmap bitmap,
                BitmapWorkerTask bitmapWorkerTask) {
            super(res, bitmap);
            bitmapWorkerTaskReference =
                new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
        }
    
        public BitmapWorkerTask getBitmapWorkerTask() {
            return bitmapWorkerTaskReference.get();
        }
    }
在执行BitmapWorkTask之前，创建一个AsyncDrawable，并且将其与ImageView绑定。

    public void loadBitmap(int resId, ImageView imageView) {
        if (cancelPotentialWork(resId, imageView)) {
            final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
            final AsyncDrawable asyncDrawable =
                    new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
            imageView.setImageDrawable(asyncDrawable);
            task.execute(resId);
        }
    }
cancelPotentialWork是一个用来检测是否有其他正在运行的后台任务和该ImageView绑定的工具函数，如果有的话将其cancel。

    public static boolean cancelPotentialWork(int data, ImageView imageView) {
        final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);
    
        if (bitmapWorkerTask != null) {
            final int bitmapData = bitmapWorkerTask.data;
            // If bitmapData is not yet set or it differs from the new data
            if (bitmapData == 0 || bitmapData != data) {
                // Cancel previous task
                bitmapWorkerTask.cancel(true);
            } else {
                // The same work is already in progress
                return false;
            }
        }
        // No task associated with the ImageView, or an existing task was cancelled
        return true;
    }
getBitmapWorkerTask()是一个用来获取和ImageView绑定的后台任务的工具函数。

    private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
       if (imageView != null) {
           final Drawable drawable = imageView.getDrawable();
           if (drawable instanceof AsyncDrawable) {
               final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
               return asyncDrawable.getBitmapWorkerTask();
           }
        }
        return null;
    }
最后一步就是在onPostExecute()函数里面监测该任务是否被cancel掉，并且和当前ImageView的绑定task一致。

    class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
        ...
    
        @Override
        protected void onPostExecute(Bitmap bitmap) {
            if (isCancelled()) {
                bitmap = null;
            }
    
            if (imageViewReference != null && bitmap != null) {
                final ImageView imageView = imageViewReference.get();
                final BitmapWorkerTask bitmapWorkerTask =
                        getBitmapWorkerTask(imageView);
                if (this == bitmapWorkerTask && imageView != null) {
                    imageView.setImageBitmap(bitmap);
                }
            }
        }
    }
这种思路可以使用于任何View有回收可能的控件，比如ListView和GridView等。
***`关键点就是解析完成的时候检测一下改线程或者task是否被cancel，并且是否属于当前ImageView`***
#3. Cache Bitmap
这一部分主要是使用内存缓存和磁盘缓存。使用LruCache和DiskLruCache即可。详细内容参考[http://developer.android.com/training/displaying-bitmaps/cache-bitmap.html](http://developer.android.com/training/displaying-bitmaps/cache-bitmap.html)或者[http://blog.csdn.net/guolin_blog/article/details/9316683](http://blog.csdn.net/guolin_blog/article/details/9316683)
需要注意的是，初始化DiskLruCache也需要进行磁盘操作，需要放入后台线程中进行。
## 处理配置的改变
常见的配置的改变就是屏幕的旋转，如果将LruCache的引用储存在Activity中，当Activity重建的时候必须重新解析图片。可以使用Fragment的setRetainInstance(true)方法使Fragment在configuration change的时候保留实例，将LruCache的引用保存在Fragment中即可避免重新建立Cache。

    private LruCache<String, Bitmap> mMemoryCache;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        RetainFragment retainFragment =
                RetainFragment.findOrCreateRetainFragment(getFragmentManager());
        mMemoryCache = retainFragment.mRetainedCache;
        if (mMemoryCache == null) {
            mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
                ... // Initialize cache here as usual
            }
            retainFragment.mRetainedCache = mMemoryCache;
        }
        ...
    }
    
    class RetainFragment extends Fragment {
        private static final String TAG = "RetainFragment";
        public LruCache<String, Bitmap> mRetainedCache;
    
        public RetainFragment() {}
    
        public static RetainFragment findOrCreateRetainFragment(FragmentManager fm) {
            RetainFragment fragment = (RetainFragment) fm.findFragmentByTag(TAG);
            if (fragment == null) {
                fragment = new RetainFragment();
                fm.beginTransaction().add(fragment, TAG).commit();
            }
            return fragment;
        }
    
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setRetainInstance(true);
        }
    }

#4 管理Bitmap内存
## 4.1 Android2.3或者更低的版本

在Android2.3或者更低的版本，当确定Bitmap不再使用的时候调用recycler()方法来释放内存，这时候就需要判断该Bitmap是否不再使用，可以使用引用计数的方法。

    private int mCacheRefCount = 0;
    private int mDisplayRefCount = 0;
    ...
    // Notify the drawable that the displayed state has changed.
    // Keep a count to determine when the drawable is no longer displayed.
    public void setIsDisplayed(boolean isDisplayed) {
        synchronized (this) {
            if (isDisplayed) {
                mDisplayRefCount++;
                mHasBeenDisplayed = true;
            } else {
                mDisplayRefCount--;
            }
        }
        // Check to see if recycle() can be called.
        checkState();
    }
    
    // Notify the drawable that the cache state has changed.
    // Keep a count to determine when the drawable is no longer being cached.
    public void setIsCached(boolean isCached) {
        synchronized (this) {
            if (isCached) {
                mCacheRefCount++;
            } else {
                mCacheRefCount--;
            }
        }
        // Check to see if recycle() can be called.
        checkState();
    }
    
    private synchronized void checkState() {
        // If the drawable cache and display ref counts = 0, and this drawable
        // has been displayed, then recycle.
        if (mCacheRefCount <= 0 && mDisplayRefCount <= 0 && mHasBeenDisplayed
                && hasValidBitmap()) {
            getBitmap().recycle();
        }
    }
    
    private synchronized boolean hasValidBitmap() {
        Bitmap bitmap = getBitmap();
        return bitmap != null && !bitmap.isRecycled();
    }

##4.2 Android 3.0之后的Bitmap内存管理
Android3.0引入了BitmapFactory.Options.inBitmap属性，如果设置了该属性，解码的时候就会尝试重用之前图片分配的内存，但是这个条件比较严苛，Android4.4之前只有两个Bitmap大小相等并且inSampleSize=1才可以，Android4.4及之后，当新的Bitmap的字节数小于等于被重用Bitmap被分配的字节数时，可以进行复用。具体的代码实现见：[http://developer.android.com/training/displaying-bitmaps/manage-memory.html#inBitmap](http://developer.android.com/training/displaying-bitmaps/manage-memory.html#inBitmap)
