# ActivityManagerService对应的BBinder是怎么来的

*com.android.server.am.ActivityManagerService*的继承关系如下：  

[ActivityManagerService][ActivityManagerServiceLink] -> [android.os.IServiceManager.Stub][IServiceManagerStubLink] -> [android.os.Binder][BinderLink]

[ActivityManagerServiceLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=417
[IServiceManagerStubLink]:https://cs.android.com/android/platform/superproject/+/master:out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/xref30/srcjars.xref/android/os/IServiceManager.java;l=111
[BinderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=584

ActivityManagerService的实例化必然伴随着超类Binder的实例化，Binder的构造器如下：  

[Binder#Binder][BinderCnstctrLink]  
&emsp;[Binder#getNativeBBinderHolder][getNativeBBinderHolderLink]  

[BinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/os/Binder.java;l=600
[getNativeBBinderHolderLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=1021

```java
    public Binder(@Nullable String descriptor)  {
        mObject = getNativeBBinderHolder();
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mObject);

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mDescriptor = descriptor;
    }
// 代码路径：frameworks/base/core/java/android/os/Binder.java
```

native的holder对象实例化之后，会在[适当的时机](./Android-Binder-2.md)实例化其中的成员变量mBinder所指向的JavaBBinder：  

[JavaBBinderHolder::get][getLink]  
&emsp;[JavaBBinder::JavaBBinder][JavaBBinderCnstctrLink]  
&emsp;&emsp;[BBinder::BBinder][BBinderCnstctrLink]  

[getLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=462
[JavaBBinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/jni/android_util_Binder.cpp;l=343
[BBinderCnstctrLink]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/binder/Binder.cpp;l=148

```c++
    sp<JavaBBinder> get(JNIEnv* env, jobject obj)
    {
        AutoMutex _l(mLock);
        sp<JavaBBinder> b = mBinder.promote();
        if (b == NULL) {
            b = new JavaBBinder(env, obj);
            if (mVintf) {
                ::android::internal::Stability::markVintf(b.get());
            }
            if (mExtension != nullptr) {
                b.get()->setExtension(mExtension);
            }
            mBinder = b;
            ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%" PRId32 "\n",
                 b.get(), b->getWeakRefs(), obj, b->getWeakRefs()->getWeakCount());
        }

        return b;
    }
// 代码路径：frameworks/base/core/jni/android_util_Binder.cpp
```

最终，我们得到的类关系图如下：  

Binder  
&emsp;&emsp;&emsp;|  
&emsp;&emsp;&emsp;|mObject  
&emsp;&emsp;&emsp;|  
JavaBBinderHolder  
&emsp;&emsp;&emsp;|  
&emsp;&emsp;&emsp;|mBinder  
&emsp;&emsp;&emsp;|  
JavaBBinder -> BBinder  
