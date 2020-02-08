### bitmap内存大小计算
首先我们将一张 491 * 492 的图片放到 `xhdpi` 目录下，然后通过 `BitmapFactory` 将图片加载到内存中
![原图](https://upload-images.jianshu.io/upload_images/7082912-0a21b32a15ceb093.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![图片信息](https://upload-images.jianshu.io/upload_images/7082912-5f075693ae3320cb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.image);
Log.e("TAG", "bitmap byte = " + bitmap.getByteCount());
Log.e("TAG", "bitmap width = " + bitmap.getWidth());
Log.e("TAG", "bitmap height = " + bitmap.getHeight());
```
根据Android默认加载四通道(ARGB_8888)图片信息可以推算该图占用的内存信息为:
```java
491 * 492 * 4 = 966288 byte
```
而控制台实际的输出为加载的bitmap实际输出为:
```java
bitmap byte = 2175624
bitmap width = 737
bitmap height = 738
737 * 738 * 4 = 2175624 byte
```
可以看到bitmap的宽高信息变大了，导致了占用的内存变大，其宽高的获取是在构造函数中，追踪 `BitmapFactory` 加载图片的源码可以发现其最终调用了`nativeDecodeStream`  方法创建了Bitmap对象:
```java
    private static Bitmap decodeStreamInternal(InputStream is, Rect outPadding, Options opts) {
        // ASSERT(is != null);
        byte [] tempStorage = null;
        if (opts != null) tempStorage = opts.inTempStorage;
        if (tempStorage == null) tempStorage = new byte[DECODE_BUFFER_SIZE];
        return nativeDecodeStream(is, tempStorage, outPadding, opts);
    }
```
该方法位于 BitmapFactory.cpp 文件中，其目录为: 
/frameworks/base/core/jni/android/graphics/BitmapFactory.cpp
```c++
static jobject nativeDecodeStream(JNIEnv* env, jobject clazz, jobject is, jbyteArray storage,
        jobject padding, jobject options) {
    ...
    if (stream.get()) {
        ...
        // 这里调用了 native 层的 方法来创建 bitmap 对象
        bitmap = doDecode(env, bufferedStream.release(), padding, options);
    }
    return bitmap;
}
```
Android8.0之后 Bitmap 改动的比较大，接下来我们分析的 native 代码基于8.0版本:
```c++
static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) {
    
    std::unique_ptr<SkStreamRewindable> streamDeleter(stream);

    // 采样率，用于图片压缩，默认=1不压缩
    int sampleSize = 1;
    // 是否只是加载图片宽高信息，默认 false
    bool onlyDecodeSize = false;
    SkColorType prefColorType = kN32_SkColorType;

    // 缩放比例 默认=1 不缩放
    float scale = 1.0f;
    bool requireUnpremultiplied = false;
    jobject javaBitmap = NULL;
    sk_sp<SkColorSpace> prefColorSpace = nullptr;

    if (options != NULL) {
	// BitmapFactory.Options 对象，用于读取 Bitmap 加载的参数信息
        sampleSize = env->GetIntField(options, gOptions_sampleSizeFieldID);
        // 默认采样率 = 1，且不能 < 1
        if (sampleSize <= 0) {
            sampleSize = 1;
        }
	// 根据 options inJustDecodeBounds 字段判断是否只加载图片尺寸
        if (env->GetBooleanField(options, gOptions_justBoundsFieldID)) {
            onlyDecodeSize = true;
        }
	
	... 初始化参数 ...

	// inScaled 参数默认支持缩放
        if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
	    // inDensity 参数 对应资源(目录)的分辨率，这里放到 xhdpi 目录 =320
            const int density = env->GetIntField(options, gOptions_densityFieldID);
	    // inTargetDensity 参数 设备dpi，我当前的测试机 = 480
	    // 以 getResources().getDisplayMetrics().densityDpi 为准，和自己计算有出入
            const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
	    // inScreenDensity 参数，默认=0
            const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
            if (density != 0 && targetDensity != 0 && density != screenDensity) {
		// 计算缩放比例，以我的测试机为例: scale = 480 / 320 = 1.5f (该值后面需要用到)
                scale = (float) targetDensity / density;
            }
        }
    }

    ...

    // Determine the output size.
    // 获取图片的原始尺寸
    SkISize size = codec->getSampledDimensions(sampleSize);

    int scaledWidth = size.width();
    int scaledHeight = size.height();
    bool willScale = false;

    // 判断是否需要对图片进行缩放
    // 这里图片的 width 和 height 都会 / simpleSize ，所以 simpleSize = 2, 则图片压缩四倍
    if (needsFineScale(codec->getInfo().dimensions(), size, sampleSize)) {
        willScale = true;
        scaledWidth = codec->getInfo().width() / sampleSize;
        scaledHeight = codec->getInfo().height() / sampleSize;
    }

    ...

    // Set the options and return if the client only wants the size.
    if (options != NULL) {	// options 默认不为空
	...
	// 将图片的width赋值到java层的options对象中
        env->SetIntField(options, gOptions_widthFieldID, scaledWidth);
	// 将图片的height赋值到java层的options对象中
        env->SetIntField(options, gOptions_heightFieldID, scaledHeight);
        env->SetObjectField(options, gOptions_mimeFieldID, mimeType);

	...
        jobject config = env->CallStaticObjectMethod(gBitmapConfig_class,
                gBitmapConfig_nativeToConfigMethodID, configID);
        env->SetObjectField(options, gOptions_outConfigFieldID, config);

        env->SetObjectField(options, gOptions_outColorSpaceFieldID,
                GraphicsJNI::getColorSpace(env, decodeColorSpace, decodeColorType));
	// options = true 则直接返回null,此时java层的 options 对象已经获取到了图片的宽高等信息
        if (onlyDecodeSize) {
            return nullptr;
        }
    }

    // Scale is necessary due to density differences.
    // 根据屏幕密度差异调整图片加载的宽高，即图片实际加载到内存的宽高
    // 上面计算得到我当前的测试机 scale = 1.5f
    if (scale != 1.0f) {
        willScale = true;
	// 1.5f * 491 + 0.5f = 737 (对应日志中的 width)
        scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);
	// 1.5f * 492 + 0.5f = 738 (对应日志中的 height)
        scaledHeight = static_cast<int>(scaledHeight * scale + 0.5f);
    }

  ... 省略代码 ...
  
    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}
```
根据 native 层的代码可以知道影响 Bitmap 加载到内存中的大小有三个因素:
1. `BitmapFactory.Options` 中的 `inSampleSize` 采样率 ,该参数由我们在 java 层中指定，默认=1；
2. 色彩格式，默认 ARGB_8888，相当于一个像素占用四个字节，RGB_565一个像素=2字节;
3. native 层计算的 `scale` ，该值由设备的 dpi 和 图片存放的目录决定，其计算方式为: 
**scale = (float) targetDensity / density**
这里 `targetDensity` 和 `density` 两者的初始化在 `BitmapFactory` 里面:
```java
public static Bitmap decodeResourceStream(Resources res, TypedValue value,
            InputStream is, Rect pad, Options opts) {
	// 保证 Options 对象不为空
        if (opts == null) {
            opts = new Options();
        }

        if (opts.inDensity == 0 && value != null) {
	    // 目录的密度，为固定值
            final int density = value.density;
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
            } else if (density != TypedValue.DENSITY_NONE) {
                opts.inDensity = density;
            }
        }
        
        if (opts.inTargetDensity == 0 && res != null) {
	    // targetDensity 即设备的 dpi
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        }
        
        return decodeStream(is, pad, opts);
}
```
`density` 的取值和图片存放的目录有关:
- ldpi(低)    ----    120dpi
- mdpi(中)  ----    160dpi
- hdpi(高)   ----    240dpi
- xhdpi(超高)  ----    320dpi
- xxhdpi(超超高)  ----    480dpi
- xxxhdpi(超超超高)  ----    640dpi

由此可以推出如果将图片放在 xxhdpi 目录下可以得到**我当前设备**的 `scale = 1`，此时图片加载的宽高为图片实际的宽高 `491 * 492` ，此时图片实际占用的内存大小为:  `491 * 492 * 4 = 966288 byte`;

### bitmap 内存开辟
上面计算完了需要输出的图片信息接下来就需要把图片文件加载到内存中，同样在 native 层的 doDecode 方法中:
```c++
static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) {
  
    ... 接上面的 图片大小计算 ...

    HeapAllocator defaultAllocator;	
    RecyclingPixelAllocator recyclingAllocator(reuseBitmap, existingBufferSize);
    ScaleCheckingAllocator scaleCheckingAllocator(scale, existingBufferSize);
    SkBitmap::HeapAllocator heapAllocator;	// 缩放图片时使用该对象
    SkBitmap::Allocator* decodeAllocator;
    if (javaBitmap != nullptr && willScale) {	// 传入复用的bitmap同时缩放图片
        // This will allocate pixels using a HeapAllocator, since there will be an extra
        // scaling step that copies these pixels into Java memory.  This allocator
        // also checks that the recycled javaBitmap is large enough.
        decodeAllocator = &scaleCheckingAllocator;
    } else if (javaBitmap != nullptr) {	// 传入复用的bitmap
        decodeAllocator = &recyclingAllocator;
    } else if (willScale || isHardware) {	// 缩放图片或者硬件加速
        // This will allocate pixels using a HeapAllocator,
        // for scale case: there will be an extra scaling step.
        // for hardware case: there will be extra swizzling & upload to gralloc step.
        decodeAllocator = &heapAllocator;
    } else {
        decodeAllocator = &defaultAllocator;
    }

    ...

    // 封装图片信息，这里的 size.width() size.height() 取的是原始图片的尺寸
    const SkImageInfo decodeInfo = SkImageInfo::Make(size.width(), size.height(),
            decodeColorType, alphaType, decodeColorSpace);

    // For wide gamut images, we will leave the color space on the SkBitmap.  Otherwise,
    // use the default.
    SkImageInfo bitmapInfo = decodeInfo;
 
    ...
    // 设置图片信息(原图)
    // 这里第一次开辟内存加载原始文件
    SkBitmap decodingBitmap;
    if (!decodingBitmap.setInfo(bitmapInfo) ||
            !decodingBitmap.tryAllocPixels(decodeAllocator, colorTable.get())) {
        // SkAndroidCodec should recommend a valid SkImageInfo, so setInfo()
        // should only only fail if the calculated value for rowBytes is too
        // large.
        // tryAllocPixels() can fail due to OOM on the Java heap, OOM on the
        // native heap, or the recycled javaBitmap being too small to reuse.
        return nullptr;
    }

    ...

    // 获取像素信息
    SkCodec::Result result = codec->getAndroidPixels(decodeInfo, decodingBitmap.getPixels(),
            decodingBitmap.rowBytes(), &codecOptions);
    
    ...

    // 需要返回的图片对象
    SkBitmap outputBitmap;
    if (willScale) {	// 输出的图片需要缩放
        // This is weird so let me explain: we could use the scale parameter
        // directly, but for historical reasons this is how the corresponding
        // Dalvik code has always behaved. We simply recreate the behavior here.
        // The result is slightly different from simply using scale because of
        // the 0.5f rounding bias applied when computing the target image size
        const float sx = scaledWidth / float(decodingBitmap.width());
        const float sy = scaledHeight / float(decodingBitmap.height());

        // Set the allocator for the outputBitmap.
        SkBitmap::Allocator* outputAllocator;
        if (javaBitmap != nullptr) {
            outputAllocator = &recyclingAllocator;
        } else {
            outputAllocator = &defaultAllocator;
        }

	...
	// 设置缩放后的图片信息
        outputBitmap.setInfo(
                bitmapInfo.makeWH(scaledWidth, scaledHeight).makeColorType(scaledColorType));
	// 重新开辟需要返回的图片的内存
        if (!outputBitmap.tryAllocPixels(outputAllocator, NULL)) {
            return nullObjectReturn("allocation failed for scaled bitmap");
        }

        SkPaint paint;
        // kSrc_Mode instructs us to overwrite the uninitialized pixels in
        // outputBitmap.  Otherwise we would blend by default, which is not
        // what we want.
        paint.setBlendMode(SkBlendMode::kSrc);
        paint.setFilterQuality(kLow_SkFilterQuality); // bilinear filtering
	// 绘制图片
        SkCanvas canvas(outputBitmap, SkCanvas::ColorBehavior::kLegacy);
        canvas.scale(sx, sy);
        canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);
    } else {
	// 不需要缩放图片直接将第一次开辟内存的图片赋值给输出对象
        outputBitmap.swap(decodingBitmap);
    }

    ...

    // 输出图片
    return bitmap::createBitmap(env, defaultAllocator.getStorageObjAndReset(),
            bitmapCreateFlags, ninePatchChunk, ninePatchInsets, -1);
}
```
这里我们不看Java层bitmap复用的情况，根据原图和输出图片的缩放情况`decodeAllocator` 会被赋值不同的对象`HeapAllocator defaultAllocator;` 和 `SkBitmap::HeapAllocator heapAllocator;` 
```c++
    SkBitmap decodingBitmap;
    if (!decodingBitmap.setInfo(bitmapInfo) ||
            !decodingBitmap.tryAllocPixels(decodeAllocator, colorTable.get())) {
        return nullptr;
    }
    ...
    // 获取像素信息
    SkCodec::Result result = codec->getAndroidPixels(decodeInfo, decodingBitmap.getPixels(),
            decodingBitmap.rowBytes(), &codecOptions);
```
这里我们分别看下Android8.0和Android7.0开辟内存的代码，其native层的Bitmap对象还是有差异的：
```c++

// region ------------------------------------ Android8.0 ------------------------------------

// /frameworks/base/core/jni/android/graphics/Graphics.cpp
bool HeapAllocator::allocPixelRef(SkBitmap* bitmap, SkColorTable* ctable) {
    mStorage = android::Bitmap::allocateHeapBitmap(bitmap, ctable);
    return !!mStorage;
}

// /frameworks/base/libs/hwui/hwui/Bitmap.cpp
static sk_sp<Bitmap> allocateBitmap(SkBitmap* bitmap, SkColorTable* ctable, AllocPixeRef alloc) {
    const SkImageInfo& info = bitmap->info();
    if (info.colorType() == kUnknown_SkColorType) {
        LOG_ALWAYS_FATAL("unknown bitmap configuration");
        return nullptr;
    }

    size_t size;

    // we must respect the rowBytes value already set on the bitmap instead of
    // attempting to compute our own.
    const size_t rowBytes = bitmap->rowBytes();
    if (!computeAllocationSize(rowBytes, bitmap->height(), &size)) {
        return nullptr;
    }
    // 创建native层的Bitmap对象，在native 内存中
    auto wrapper = alloc(size, info, rowBytes, ctable);
    if (wrapper) {
        wrapper->getSkBitmap(bitmap);
        // since we're already allocated, we lockPixels right away
        // HeapAllocator behaves this way too
        bitmap->lockPixels();
    }
    return wrapper;
}

// endregion ---------------------------------------------------------------------------------


// region ------------------------------------ Android7.0 ------------------------------------

// /frameworks/base/core/jni/android/graphics/Graphics.cpp
bool JavaPixelAllocator::allocPixelRef(SkBitmap* bitmap, SkColorTable* ctable) {
    JNIEnv* env = vm2env(mJavaVM);

    mStorage = GraphicsJNI::allocateJavaPixelRef(env, bitmap, ctable);
    return mStorage != nullptr;
}

// /frameworks/base/core/jni/android/graphics/Graphics.cpp
android::Bitmap* GraphicsJNI::allocateJavaPixelRef(JNIEnv* env, SkBitmap* bitmap,
                                             SkColorTable* ctable) {
    const SkImageInfo& info = bitmap->info();
    if (info.colorType() == kUnknown_SkColorType) {
        doThrowIAE(env, "unknown bitmap configuration");
        return NULL;
    }

    size_t size;
    if (!computeAllocationSize(*bitmap, &size)) {
        return NULL;
    }

    // we must respect the rowBytes value already set on the bitmap instead of
    // attempting to compute our own.
    const size_t rowBytes = bitmap->rowBytes();
    // 调用java代码创建数组
    jbyteArray arrayObj = (jbyteArray) env->CallObjectMethod(gVMRuntime,
                                                             gVMRuntime_newNonMovableArray,
                                                             gByte_class, size);
    if (env->ExceptionCheck() != 0) {
        return NULL;
    }
    SkASSERT(arrayObj);
    jbyte* addr = (jbyte*) env->CallLongMethod(gVMRuntime, gVMRuntime_addressOf, arrayObj);
    if (env->ExceptionCheck() != 0) {
        return NULL;
    }
    SkASSERT(addr);
    // 将 native 层的 Bitmap 对象开辟在Java内存中
    android::Bitmap* wrapper = new android::Bitmap(env, arrayObj, (void*) addr,
            info, rowBytes, ctable);
    wrapper->getSkBitmap(bitmap);
    // since we're already allocated, we lockPixels right away
    // HeapAllocator behaves this way too
    bitmap->lockPixels();

    return wrapper;
}

// /frameworks/base/core/jni/android/graphics/Bitmap.cpp
Bitmap::Bitmap(JNIEnv* env, jbyteArray storageObj, void* address,
            const SkImageInfo& info, size_t rowBytes, SkColorTable* ctable)
        : mPixelStorageType(PixelStorageType::Java) {
    env->GetJavaVM(&mPixelStorage.java.jvm);
    mPixelStorage.java.jweakRef = env->NewWeakGlobalRef(storageObj);
    mPixelStorage.java.jstrongRef = nullptr;
    mPixelRef.reset(new WrappedPixelRef(this, address, info, rowBytes, ctable));
    // Note: this will trigger a call to onStrongRefDestroyed(), but
    // we want the pixel ref to have a ref count of 0 at this point
    mPixelRef->unref();
}

// endregion ---------------------------------------------------------------------------------

```
接着系统会调用`tryAllocPixels`方法**第一次**开辟内存加载原始图片,之所以会强调**第一次**开辟内存是因为其开辟的内存大小是原始图片的大小，如果图片如果需要缩放，即`if (willScale) `这个条件进来则系统会再次调用 `tryAllocPixels` 方法再次开辟输出文件的内存。
Android7.0和Android8.0的差异在于native层的Bitmap对象所在的内存是Java虚拟机中还是native中，native的堆内存比JVM的内存要大，其内存回收的机制也不一样；
最后 native 层调用了 Bitmap.cpp 的 `createBitmap` 方法创建一个 Java 层的 Bitmap对象返回;
```c++
jobject createBitmap(JNIEnv* env, Bitmap* bitmap,
        int bitmapCreateFlags, jbyteArray ninePatchChunk, jobject ninePatchInsets,
        int density) {
    ...
    // jni层创建Java层Bitmap对象
    jobject obj = env->NewObject(gBitmap_class, gBitmap_constructorMethodID,
            reinterpret_cast<jlong>(bitmapWrapper), bitmap->width(), bitmap->height(), density,
            isMutable, isPremultiplied, ninePatchChunk, ninePatchInsets);
    ...
    return obj;
}
```
