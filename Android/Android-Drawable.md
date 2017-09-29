Drawable 是一个抽象类，提供了一些 API 方法去处理各种资源的绘制，但是又不具备 View 的事件与交互处理能力。

Drawable表示一种图像的概念，通常被用来作为VIew的背景来使用，Drawable会被拉伸至与VIew同等大小。Drawable的内部宽高通过getIntrinsicWidth和getIntrinsicHeight这两个方法获取，并不是所有的Drawable都有内部的宽高。

Drawable的实际区域大小可以通过getBounds方法来得到，一般跟View的大小相同。

官方文档：

https://developer.android.google.cn/guide/topics/resources/drawable-resource.html

## BitmapDrawable

表示的是一张图片，对应于`<bitmap>`标签。

```xml
<?xml version="1.0" encoding="utf-8"?>
<bitmap
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:src="@[package:]drawable/drawable_resource"
    android:antialias=["true" | "false"]
    android:dither=["true" | "false"]
    android:filter=["true" | "false"]
    android:gravity=["top" | "bottom" | "left" | "right" | "center_vertical" |
                      "fill_vertical" | "center_horizontal" | "fill_horizontal" |
                      "center" | "fill" | "clip_vertical" | "clip_horizontal"]
    android:mipMap=["true" | "false"]
    android:tileMode=["disabled" | "clamp" | "repeat" | "mirror"] />
```

antialias:开启抗锯齿功能，让图片变得更平滑。

dither：当图片像素配置与手机屏幕的像素配置不一致时，可以让高质量图片在低质量的屏幕上还能保持较好的显示效果。

filter：当图片尺寸被拉伸或压缩时，开启过滤功能可以获得更好的显示效果。

titleMode：mirror在水平和竖直方向上的镜面投影效果；clamp图片四周的像素会扩展到周围区域。



## ShapeDrawable

通过颜色来构造图像，既可以是纯色的图形，可以是具有渐变效果的图形，对应与`<shape>`标签。

```xml
<?xml version="1.0" encoding="utf-8"?>
<shape
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape=["rectangle" | "oval" | "line" | "ring"] >
    <corners
        android:radius="integer"
        android:topLeftRadius="integer"
        android:topRightRadius="integer"
        android:bottomLeftRadius="integer"
        android:bottomRightRadius="integer" />
    <gradient
        android:angle="integer"
        android:centerX="float"
        android:centerY="float"
        android:centerColor="integer"
        android:endColor="color"
        android:gradientRadius="integer"
        android:startColor="color"
        android:type=["linear" | "radial" | "sweep"]
        android:useLevel=["true" | "false"] />
    <padding
        android:left="integer"
        android:top="integer"
        android:right="integer"
        android:bottom="integer" />
    <size
        android:width="integer"
        android:height="integer" />
    <solid
        android:color="color" />
    <stroke
        android:width="integer"
        android:color="color"
        android:dashWidth="integer"
        android:dashGap="integer" />
</shape>
```



## LayerDrawable

对应于`<layer-list>`标签，表示一种层次化的Drawable集合。

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list
    xmlns:android="http://schemas.android.com/apk/res/android" >
    <item
        android:drawable="@[package:]drawable/drawable_resource"
        android:id="@[+][package:]id/resource_name"
        android:top="dimension"
        android:right="dimension"
        android:bottom="dimension"
        android:left="dimension" />
</layer-list>
```



## StateListDrawable

对应于`<selector>`标签，常用于Button的Selector。

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android"
    android:constantSize=["true" | "false"]
    android:dither=["true" | "false"]
    android:variablePadding=["true" | "false"] >
    <item
        android:drawable="@[package:]drawable/drawable_resource"
        android:state_pressed=["true" | "false"]
        android:state_focused=["true" | "false"]
        android:state_hovered=["true" | "false"]
        android:state_selected=["true" | "false"]
        android:state_checkable=["true" | "false"]
        android:state_checked=["true" | "false"]
        android:state_enabled=["true" | "false"]
        android:state_activated=["true" | "false"]
        android:state_window_focused=["true" | "false"] />
</selector>
```



[Android 应用层开发 Drawable 的一些叨叨絮](http://blog.csdn.net/yanbober/article/details/56844869)

