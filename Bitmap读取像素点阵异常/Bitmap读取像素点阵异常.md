# Android开发笔记-记一次读取Bitmap像素点阵异常

## 前言

该问题发现于MI8 LITE MIUI 11.0.2 Android版本10  
本文章源码版本皆为Android10版本

在一次对iOS客户端生成的图片进行二值处理的过程中，发现使用Bitmap原生的获取图片像素点阵获取到的像素点全是0xFF000000，因此导致了图片全黑或全白的问题。

本篇将记录问题出现的可能原因与相关的解决方案

## 原因分析

我们先从Bitmap自带的获取像素点阵函数入手：
```java

    /**
     * ......
     *
     * @param pixels   接收拷贝值的数组
     * @param offset   像素起始位置的偏移量
     * @param stride   要在每行之间跳过的像素数量，必须大于bitmap的宽度
     * @param x        读取的第一个像素的起始X坐标值
     * @param y        读取的第一个像素的起始Y坐标值
     * @param width    每行宽度
     * @param height   每列高度
     * 
     * ......
     */
    public void getPixels(@ColorInt int[] pixels, int offset, int stride,
                          int x, int y, int width, int height) {
        // Bitmap的相关属性检查
        checkRecycled("Can't call getPixels() on a recycled bitmap");
        checkHardware("unable to getPixels(), "
                + "pixel access is not supported on Config#HARDWARE bitmaps");
        if (width == 0 || height == 0) {
            return; // nothing to do
        }

        // 检查指定的参数是否可用
        // 若不可用将抛出ArrayIndexOutOfBoundsException异常
        checkPixelsAccess(x, y, width, height, offset, stride, pixels);

        // 调用native方法实现像素数组的拷贝
        nativeGetPixels(mNativePtr, pixels, offset, stride,
                        x, y, width, height);
    }
```
根据方法的相关注释与源码我们可以了解到，这个方法在通常情况下会将图片非预乘的ARGB颜色值拷贝至pixels数组中，使用的色彩空间为`ColorSpace.Named#SRGB` sRGB色彩空间  

那么进一步深入[系统源码](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/android10-release/core/jni/android/graphics/Bitmap.cpp#965)

```cpp
static void Bitmap_getPixels(JNIEnv* env, jobject, jlong bitmapHandle,
        jintArray pixelArray, jint offset, jint stride,
        jint x, jint y, jint width, jint height) {
    SkBitmap bitmap;
    reinterpret_cast<BitmapWrapper*>(bitmapHandle)->getSkBitmap(&bitmap);
    auto sRGB = SkColorSpace::MakeSRGB();
    SkImageInfo dstInfo = SkImageInfo::Make(
            width, height, kBGRA_8888_SkColorType, kUnpremul_SkAlphaType, sRGB);
    jint* dst = env->GetIntArrayElements(pixelArray, NULL);
    bitmap.readPixels(dstInfo, dst + offset, stride * 4, x, y);
    env->ReleaseIntArrayElements(pixelArray, dst, 0);
}
```
可以看出native是调用了skia库进行图片像素值拷贝，skia拷贝时根据指定的SkImageInfo对图片的颜色空间进行转换后拷贝至指定的数据空间。  
由于对于skia库的具体实现不了解，源码查证至此结束。具体可参见[skia库的相关源码](https://skia.googlesource.com/skia/+/refs/heads/android/10-release/src/core/)  

根据上述源码分析，猜测问题应该发生在skia库进行色彩空间转换中，相关的方法为`SkConvertPixels::convert_with_pipeline`，至此开始着手尝试解决方案。

## 解决方案
在Android中最为常用的图片格式为JPEG与PNG，为了保留透明通道与图片质量，使用`Bitmap.Config.ARGB_8888`模式加载图片。  
根据原因分析我们基本确定问题出现在bitmap自带的拷贝方法，那么最快的做法就是绕过bitmap的方法进行像素数组的拷贝。  
由于Bitmap中实现的拷贝像素数组相关的方法都是通过skia库实现的，那么使用起来显然会出现同样的问题，在需要保证性能与效果的情况下，最合适的方案就是使用jni来实现拷贝。  

根据查询资料与阅读源码，可以看到在[bitmap.cpp](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/android10-release/core/jni/android/graphics/Bitmap.cpp#270)中的`lockPixels(JNIEnv* env, jobject bitmap)`方法，该方法的作用是锁定当前bitmap的像素数组，并返回一个指向数组的指针。
在拿到原像素数组的指针后需要做的事情就很简单了。由于我们图片加载的模式已经是`Bitmap.Config.ARGB_8888`，那么只需要直接将数组内容拷贝一份并返回即可，实现如下：

```cpp
#include <android/bitmap.h>

getBitmapPixels(JNIEnv *env, jclass clazz, jobject bitmap) {
    AndroidBitmapInfo info;
    if (AndroidBitmap_getInfo(env, bitmap, &info) < 0) {
        LOGE("cannot load bitmap info");
        return (*env).NewIntArray(0);
    }
    void *address;
    if (AndroidBitmap_lockPixels(env, bitmap, &address) < 0) {
        LOGE("cannot load bitmap pixels");
    }
    int *src = (int *) address;

    jint length = info.width * info.height;
    jintArray array = env->NewIntArray(length);
    // 拷贝数组
    env->SetIntArrayRegion(array, 0, length, src);
    // 在拷贝完数组需要释放当前图片像素数组的锁
    AndroidBitmap_unlockPixels(env, bitmap);
    return array;
}
```

## 总结

本文描述的问题仅出现在Android 10版本中，出现问题是还以为是Android 10版本调整了部分API的问题，在一步步调试与排查后发现了问题在拷贝像素数组上。随后，在经过对比Android 9源码发现Android 10在这部分上确实进行了一部分修改，但由于对于skia库并没有深入研究，所以不对实际问题进行更深入的分析。  

文中相关解决方案在公司项目测试后没有再出现该问题，如果有更多解决方案或文中有错误欢迎指出与讨论。  
<b> 联系我 <yuhanglee1996@gmail.com> </b>

## 参考文章
[Android Bitmap像素排列与JNI操作](https://segmentfault.com/a/1190000023731917)