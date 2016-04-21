---
layout:     post
title:      "Example Post"
subtitle:   "ES5, ES6, ES2016, ES.Next: What's going on with JavaScript versioning?"
date:       2015-09-22
author:     "Hux"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 前端开发
    - JavaScript
    - 翻译
---


#### 著作权声明

本文译自 [ES5, ES6, ES2016, ES.Next: What's going on with JavaScript versioning?](http://benmccormick.org/2015/09/14/es5-es6-es2016-es-next-whats-going-on-with-javascript-versioning/)   
译者 [黄玄](http://weibo.com/huxpro)，首次发布于 [Hux Blog](http://huangxuan.me)，转载请保留以上链接
hahahahahhaa

## 高效加载大图片
Android在加载大图片时，由于程序占用大量内存，内存不足时很容易发生OOM(OutOfMemory)异常。

如果想要知道每个应用程序最高可用内存是多少，可以使用如下代码：
```java 
    int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);  
    Log.d("TAG", "Max memory is " + maxMemory + "KB");  
```
那内存不足时我们要怎样才能加载大图片呢？

显然我们要对大图片进行压缩处理，下面就简单介绍常用的压缩图片方法：

BitmapFactory这个类提供了多个解析方法(decodeByteArray, decodeFile, decodeResource等)用于创建Bitmap对象，我们应该根据图片的来源选择合适的方法。比如SD卡中的图片可以使用decodeFile方法，网络上的图片可以使用decodeStream方法，资源文件中的图片可以使用decodeResource方法。这些方法会尝试为已经构建的bitmap分配内存，这时就会很容易导致OOM出现。为此每一种解析方法都提供了一个可选的BitmapFactory.Options参数，将这个参数的inJustDecodeBounds属性设置为true就可以让解析方法禁止为bitmap分配内存，返回值也不再是一个Bitmap对象，而是null。虽然Bitmap是null了，但是BitmapFactory.Options的outWidth、outHeight和outMimeType属性都会被赋值。这个技巧让我们可以在加载图片之前就获取到图片的长宽值和MIME类型，从而根据情况对图片进行压缩。

本文仅采用资源文件中的图片作为例子，所以在解析图片时我们只需采用decodeResource方法即可。decodeResource方法返回的对象为Bitmap，接受三个参数，分别为包含图片的资源文件对象，

##### 得到初始图片大小
在图片压缩之前，我们需要知道初始图片的大小，代码如下：
```java
 final BitmapFactory.Options options = new BitmapFactory.Options();  
    options.inJustDecodeBounds = true;  
    BitmapFactory.decodeResource(res, resId, options);  
    // int height = options.outHeight;
    // int width = options.outWidth;
```
第二行代码中Options对象的inJustDecodeBounds的值设为true时，表示下面的decodeResource只关心图像边界（事实上还有MIME类型,不过不影响）,而对图像边界内的内容不关心，所以inJustDecodeBounds方法设为true时不会发生OOM异常。解析后图像的高度和宽度分别存储在Options对象的outHeight和outWidth中，这就实现了初始图片大小的提取。

##### 对图片进行压缩
对图片的压缩，通过设置BitmapFactory.Options中inSampleSize的值就可以实现。

比如我们有一张2048 * 1536像素的图片，将inSampleSize的值设置为4，就可以把这张图片压缩成512 * 384像素。原本加载这张图片需要占用13M的内存，压缩后就只需要占用0.75M了(假设图片是ARGB_8888类型，即每个像素点占用4个字节)。下面的方法可以根据传入的宽和高，计算出合适的inSampleSize值：
```java
public static int calculateInSampleSize(BitmapFactory.Options options,  
        int reqWidth, int reqHeight) {  
    // 源图片的高度和宽度  
    final int height = options.outHeight;  
    final int width = options.outWidth;  
    int inSampleSize = 1;  
    if (height > reqHeight || width > reqWidth) {  
        // 计算出实际宽高和目标宽高的比率  
        final int heightRatio = Math.round((float) height / (float) reqHeight);  
        final int widthRatio = Math.round((float) width / (float) reqWidth);  
        // 选择宽和高中最小的比率作为inSampleSize的值，这样可以保证最终图片的宽和高  
        // 一定都会大于等于目标的宽和高。  
        inSampleSize = heightRatio < widthRatio ? heightRatio : widthRatio;  
    }  
    return inSampleSize;  
}  
```

##### 实践
将一个大小为4.57MB的图像test.jpg导入到drawable中,分别采用有压缩图片和无压缩图片的方法分别对图片进行显示
```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ImageView mImageView = (ImageView)findViewById(R.id.myimage);
        //ImageView mImageView2 = (ImageView) findViewById(R.id.myimage2);
        mImageView.setImageBitmap(decodeSampledBitmapFromresource(getResources(),R.drawable.test,100,100));
        //mImageView2.setImageResource(R.drawable.test);

    }
    //计算压缩比例
    public static int calculateInSampleSize(BitmapFactory.Options options,int reqWidth,int reqHeight){
        final int height = options.outHeight;
        final int width = options.outWidth;
        int inSampleSize = 1;
        if (height>reqHeight||width>reqWidth){
            final int heightRatio = Math.round((float)height/(float)reqHeight);
            final int widthRatio = Math.round((float)width/(float)reqWidth);
            inSampleSize = heightRatio < widthRatio ? heightRatio : widthRatio;
        }
        return inSampleSize;

    }
//核心代码
    public static Bitmap decodeSampledBitmapFromresource(Resources res,int resId,int reqWidth,int reqHeight){
        //获取初始图片大小
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(res,resId,options);
        //获得该图片的压缩比例
        options.inSampleSize = calculateInSampleSize(options,reqWidth,reqHeight);
        //重新对图片进行解析
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(res,resId,options);
    }
}
```
![java-javascript](/img/in-post/post-avoidOOM/test0.png)
<small class="img-hint">有压缩图片的结果</small>

![java-javascript](/img/in-post/post-avoidOOM/test1.png)
<small class="img-hint">无压缩图片的结果</small>

我们可以发现在有压缩图片的情况时界面正常显示，而在无压缩图片的情况时，程序发生错误。

我们再看看编译器报的错误
![java-javascript](/img/in-post/post-avoidOOM/result.png)
<small class="img-hint">程序发生了OOM异常</small>

我们发现程序发生OutOfMemoryError异常！这说明由于读取图片时占用内存过大，导致应用无法分配内存，这充分说明了研究图片压缩的重要性！


