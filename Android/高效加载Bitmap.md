## Bitmap

一张png或jpg图片经过解码后会变成Bitmap对象，它会将图片上的每一个像素都保存在内存中，导致稍有不慎就会创建出一个占用内存非常大的Bitmap对象，从而导致加载过慢，还会有内存溢出的风险。

BitmapFactory类提供了4类方法：decodeFile，decodeResource，decodeStream和decodeByteArray来加载一个Bitmap对象。

### decodeResource与decodeFile

**decodeFile()**用于读取SD卡上的图，得到的是图片的原始尺寸 
**decodeResource()**用于读取Res、Raw等资源，得到的是图片的原始尺寸 * 缩放系数

**缩放系数的计算**

通过BitmapFactory.Options的这几个参数可以调整缩放系数。

```java
public class BitmapFactory {
    public static class Options {
        public boolean inScaled;     // 默认true
        public int inDensity;        // 无dpi的文件夹下默认160
        public int inTargetDensity;  // 取决具体屏幕
    }
}
```

**inScaled属性**

如果inScaled设置为false，则不进行缩放，解码后的图片与原图一致。

如果inScaled设置为true，则根据inDensity和inTargetDensity计算缩放系数。

计算公式为，缩放系数=inTargetDensity/inDensity

放在不同的dpi文件夹会影响inDensity，进而影响加载出来图片的大小。

### Bitmap占用的内存

一张图片Bitmap所占用的内存 = 图片长度 x 图片宽度 x 一个像素点占用的字节数

而Bitmap.Config，正是指定单位像素占用的字节数的重要参数。

ALPHA_8 
表示8位Alpha位图,即A=8,一个像素点占用1个字节,它没有颜色,只有透明度 
ARGB_4444 
表示16位ARGB位图，即A=4,R=4,G=4,B=4,一个像素点占4+4+4+4=16位，2个字节 
ARGB_8888 
表示32位ARGB位图，即A=8,R=8,G=8,B=8,一个像素点占8+8+8+8=32位，4个字节 
RGB_565 
表示16位RGB位图,即R=5,G=6,B=5,它没有透明度,一个像素点占5+6+5=16位，2个字节

## 先压缩后加载

根据传入的宽和高（ImageView），通过设置BitmapFactory.Options中inSampleSize的值就可以实现可以实现对图片的压缩。当inSampleSize为1时，采样后的图片大小与原图一致；当i你sampleSize大小为2时，那么采样后的图片其宽高均为原图大小的二分之一，像素数为原图的四分之一。因此，inSampleSize大于一才有缩小效果，缩放比例为1/(inSampleSize的2次方)。

具体流程如下：

1. 将BitmapFactory.Options中的inJustDecodeBounds参数设为true，准备解析图片的原始宽高信息。
2. 从BitmapFactory.Options中去除图片的原始宽高信息
3. 根据采样率规则结合目标View尺寸大小计算出采样率inSampleSize
4. 将BitmapFactory.Options的inJustDecodeBounds参数设为false，然后重新加载图片。

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
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,  
        int reqWidth, int reqHeight) {  
    // 第一次解析将inJustDecodeBounds设置为true，来获取图片大小  
    final BitmapFactory.Options options = new BitmapFactory.Options();  
    options.inJustDecodeBounds = true;  
    BitmapFactory.decodeResource(res, resId, options);  
    // 调用上面定义的方法计算inSampleSize值  
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);  
    // 使用获取到的inSampleSize值再次解析图片  
    options.inJustDecodeBounds = false;  
    return BitmapFactory.decodeResource(res, resId, options);  
}  
// 加载图片
mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
```

## Cache缓存

### LruCache

```java
    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {  
        @Override  
        protected int sizeOf(String key, Bitmap bitmap) {  
            // 重写此方法来衡量每张图片的大小，默认返回图片数量。  
            return bitmap.getByteCount() / 1024;  
        }  
    };  
```

LruCache内部维护一个LinkedHashMap以**强引用**的方式存储外界得缓存对象。

不同引用得区别如下：

- 软引用：当一个对象只有软引用存在时，系统内存不足时该对象可能会被gc
- 弱引用：当一个对象只有弱引用存在时，此对象随时会被gc回收

关于LruCache相关总结：

- LruCache 是通过 LinkedHashMap 构造方法的第三个参数的 `accessOrder=true` 实现了 `LinkedHashMap` 的数据排序**基于访问顺序** （最近访问的数据会在链表尾部），在容量溢出的时候，将链表头部的数据移除。从而，实现了 LRU 数据缓存机制。
- LruCache 在内部的get、put、remove包括 trimToSize 都是线程安全的（因为都上锁了）。
- LruCache 自身并没有释放内存，将 LinkedHashMap 的数据移除了，如果数据还在别的地方被引用了，还是有泄漏问题，还需要手动释放内存。
- 覆写 `entryRemoved` 方法能知道 LruCache 数据移除是是否发生了冲突，也可以去手动释放资源。
- maxSize` 和 `sizeOf(K key, V value)` 方法的覆写息息相关，必须相同单位。（ 比如 maxSize 是7MB，自定义的 sizeOf 计算每个数据大小的时候必须能算出与MB之间有联系的单位 ）

每次调用LinkedHashMap的get方法时，如果`accessOrder=true` ，那么它会调用`makeTail`方法，将entry移到链表的尾部。

参考LruCache的源码解析：

https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/LruCache%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md

### DiskLruCache

http://blog.csdn.net/guolin_blog/article/details/28863651

## 大图加载

通过BitmapRegionDecoder这个类实现按区域加载Bitmap。



## 参考

- [Android Bitmap 知识点梳理](http://blog.csdn.net/u012124438/article/details/72614602)