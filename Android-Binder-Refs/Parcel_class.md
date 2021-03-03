# Parcel类的有关说明

[Parcel#obtain][ParcelObtainLink]  
&emsp;[Parcel#Parcel][ParcelParcelLink]  
&emsp;&emsp;[Parcel#init][ParcelinitLink]  
&emsp;&emsp;&emsp;[Parcel#nativeCreate][ParcelnativeCreateLink]  
&emsp;&emsp;&emsp;&emsp;[ParcelRef::create][ParcelRefCreateLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[ParcelRef::ParcelRef][ParcelRefCnstctrLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::Parcel][ParcelCnstctrLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::initState][ParcelInitStateLink]  

```c++
void Parcel::initState()
{
    LOG_ALLOC("Parcel %p: initState", this);
    mError = NO_ERROR;
    mData = nullptr;  // Note: uint8_t* mData; Parcel中包装的所有数据所在内存的首地址
    mDataSize = 0;    // Note: size_t mDataSize; Parcel当前已有数据的大小，单位为size_t
    mDataCapacity = 0;// Note: size_t mDataCapacity; Parcel中当前数据的容量，单位为size_t
    mDataPos = 0;     // Note: mutable size_t mDataPos; Parcel中已有数据相对于首地址的偏移量，(mData+mDataPos)也就是将要添加的新数据的首地址
    ALOGV("initState Setting data size of %p to %zu", this, mDataSize);
    ALOGV("initState Setting data pos of %p to %zu", this, mDataPos);
    mObjects = nullptr;
    mObjectsSize = 0;
    mObjectsCapacity = 0;
    mNextObjectHint = 0;
    mObjectsSorted = false;
    mHasFds = false;
// ... 省略代码
}
// 代码路径：frameworks/native/libs/binder/Parcel.cpp
```

[ParcelObtainLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=424
[ParcelParcelLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=3510
[ParcelinitLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=3518
[ParcelnativeCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=531
[ParcelRefCreateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/include/binder/ParcelRef.h;l=33
[ParcelRefCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/include/binder/ParcelRef.h;l=38
[ParcelCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=268
[ParcelInitStateLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2902

到此为止，Parcel类中没有任何数据，mData，mDataSize，mDataCapacity，mDataPos是0/空。  

[Parcel#writeInterfaceToken][writeInterfaceTokenLink]  
&emsp;[Parcel#nativeWriteInterfaceToken][nativeWriteInterfaceTokenLink]  
&emsp;&emsp;[Parcel::writeInterfaceToken][nParcelWriteInterfaceTokenLink]  
&emsp;&emsp;&emsp;[Parcel::writeInt32][nParcelWriteInt32Link]  
&emsp;&emsp;&emsp;&emsp;[Parcel::writeAligned][nParcelWriteAlignedLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::growData][nParcelGrowDataLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::continueWrite][nParcelContinueWriteLink]  
&emsp;&emsp;&emsp;&emsp;&emsp;[Parcel::finishWrite][nParcelfinishWriteLink]  

```c++
template<class T>
status_t Parcel::writeAligned(T val) {
    static_assert(PAD_SIZE_UNSAFE(sizeof(T)) == sizeof(T));

    if ((mDataPos+sizeof(val)) <= mDataCapacity) {
restart_write:
        *reinterpret_cast<T*>(mData+mDataPos) = val;
// Note: *(reinterpret_cast<T*>(mData+mDataPos)) = val;
        return finishWrite(sizeof(val));
    }

    status_t err = growData(sizeof(val));
    if (err == NO_ERROR) goto restart_write;
    return err;
}

status_t Parcel::growData(size_t len)
{
    if (len > INT32_MAX) {
        // don't accept size_t values which may have come from an
        // inadvertent conversion from a negative int.
        return BAD_VALUE;
    }

    if (len > SIZE_MAX - mDataSize) return NO_MEMORY; // overflow
    if (mDataSize + len > SIZE_MAX / 3) return NO_MEMORY; // overflow
    size_t newSize = ((mDataSize+len)*3)/2;
    return (newSize <= mDataSize)
            ? (status_t) NO_MEMORY
            : continueWrite(std::max(newSize, (size_t) 128));
}

status_t Parcel::continueWrite(size_t desired)
{
    if (desired > INT32_MAX) {
// ... 省略代码
    }

    // If shrinking, first adjust for any objects that appear
    // after the new data size.
    size_t objectsSize = mObjectsSize;
    if (desired < mDataSize) {
// ... 省略代码
    }

    if (mOwner) {
// ... 省略代码
    } else if (mData) {
// ... 省略代码
    } else {
        // This is the first data.  Easy!
        uint8_t* data = (uint8_t*)malloc(desired);
        if (!data) {
// ... 省略代码
        }

        if(!(mDataCapacity == 0 && mObjects == nullptr
// ... 省略代码
        }

        LOG_ALLOC("Parcel %p: allocating with %zu capacity", this, desired);
        gParcelGlobalAllocSize += desired;
        gParcelGlobalAllocCount++;

        mData = data;
        mDataSize = mDataPos = 0;
        ALOGV("continueWrite Setting data size of %p to %zu", this, mDataSize);
        ALOGV("continueWrite Setting data pos of %p to %zu", this, mDataPos);
        mDataCapacity = desired;
    }

    return NO_ERROR;
}
// 代码路径：frameworks/native/libs/binder/Parcel.cpp
```

[writeInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Parcel.java;l=656
[nativeWriteInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_os_Parcel.cpp;l=688
[nParcelWriteInterfaceTokenLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=535
[nParcelWriteInt32Link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=967
[nParcelWriteAlignedLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=1594
[nParcelGrowDataLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2657
[nParcelContinueWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=2740
[nParcelfinishWriteLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Parcel.cpp;l=642
