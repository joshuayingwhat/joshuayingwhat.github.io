---
layout: post
title:  "九:binder框架framework分析"
categories: android
tags:  binder
---

* content
{:toc}

![](https://s2.loli.net/2024/07/23/7wYmXtxjBlnJOZA.jpg)
<font color="red">framework binder就是将bpbinder转换为binderproxy使用。</font>

# 注册服务过程 #
servicemanager.java
```java
public static void addService(String name, IBinder service, boolean allowIsolated) {
    try {
        //先获取SMP对象，则执行注册服务操作【见小节3.2/3.4】
        getIServiceManager().addService(name, service, allowIsolated);
    } catch (RemoteException e) {
        Log.e(TAG, "error in addService", e);
    }
}
```
......

### getIServiceManager ###
```java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }
    //【分别见3.2.1和3.3】
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
```
采用单利模式获取sServiceManager，返回servicemanagerproxy。
### getContextObject ###
```java
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);  //【见3.2.2】
}
```
就是通过handle=0去获取servicemanager的引用

### javaObjectForIBinder

```java
jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val) {
    if (val == NULL) return NULL;

    if (val->checkSubclass(&gBinderOffsets)) { //返回false
        jobject object = static_cast<JavaBBinder*>(val.get())->object();
        return object;
    }

    AutoMutex _l(mProxyLock);

    jobject object = (jobject)val->findObject(&gBinderProxyOffsets);
    if (object != NULL) { //第一次object为null
        jobject res = jniGetReferent(env, object);
        if (res != NULL) {
            return res;
        }
        android_atomic_dec(&gNumProxyRefs);
        val->detachObject(&gBinderProxyOffsets);
        env->DeleteGlobalRef(object);
    }

    //创建BinderProxy对象
    object = env->NewObject(gBinderProxyOffsets.mClass, gBinderProxyOffsets.mConstructor);
    if (object != NULL) {
        //BinderProxy.mObject成员变量记录BpBinder对象
        env->SetLongField(object, gBinderProxyOffsets.mObject, (jlong)val.get());
        val->incStrong((void*)javaObjectForIBinder);

        jobject refObject = env->NewGlobalRef(
                env->GetObjectField(object, gBinderProxyOffsets.mSelf));
        //将BinderProxy对象信息附加到BpBinder的成员变量mObjects中
        val->attachObject(&gBinderProxyOffsets, refObject,
                jnienv_to_javavm(env), proxy_cleanup);

        sp<DeathRecipientList> drl = new DeathRecipientList;
        drl->incStrong((void*)javaObjectForIBinder);
        //BinderProxy.mOrgue成员变量记录死亡通知对象
        env->SetLongField(object, gBinderProxyOffsets.mOrgue, reinterpret_cast<jlong>(drl.get()));

        android_atomic_inc(&gNumProxyRefs);
        incRefsCreated(env);
    }
    return object;
}
```

根据BpBinder(C++)生成BinderProxy(Java)对象. 主要工作是创建BinderProxy对象,并把BpBinder对象地址保存到BinderProxy.mObject成员变量. 到此，可知ServiceManagerNative.asInterface(BinderInternal.getContextObject()) 等价于

```java
ServiceManagerNative.asInterface(new BinderProxy());
```

### ServiceManagerNative.asInterface

```java
 static public IServiceManager asInterface(IBinder obj)
{
    if (obj == null) { //obj为BpBinder
        return null;
    }
    //由于obj为BpBinder，该方法默认返回null
    IServiceManager in = (IServiceManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    return new ServiceManagerProxy(obj); //【见小节3.3.1】
}
```

## servicemanagerproxy

```java
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }
}
```

mremote本质就是binderproxy(bpbinder)此时的mremote就是handle=0的servicemamager。

### addservice

```java
public void addService(String name, IBinder service, boolean allowIsolated) throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    //【见小节3.5】
    data.writeStrongBinder(service);
    data.writeInt(allowIsolated ? 1 : 0);
    //mRemote为BinderProxy【见小节3.7】
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    reply.recycle();
    data.recycle();
}
```

### writeStrongBinder

```java
public writeStrongBinder(IBinder val){
    //此处为Native调用【见3.5.1】
    nativewriteStrongBinder(mNativePtr, val);
}
```

最终调用了**android_os_Parcel_writeStrongBinder方法**

### **android_os_Parcel_writeStrongBinder**

```java
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object) {
    //将java层Parcel转换为native层Parcel
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        //【见3.5.2】
        const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
        if (err != NO_ERROR) {
            signalExceptionForError(env, clazz, err);
        }
    }
}
```

### ibinderForJavaObject

```java
sp<IBinder> ibinderForJavaObject(JNIEnv* env, jobject obj)
{
    if (obj == NULL) return NULL;

    //Java层的Binder对象
    if (env->IsInstanceOf(obj, gBinderOffsets.mClass)) {
        JavaBBinderHolder* jbh = (JavaBBinderHolder*)
            env->GetLongField(obj, gBinderOffsets.mObject);
        return jbh != NULL ? jbh->get(env, obj) : NULL; //【见3.5.3】
    }
    //Java层的BinderProxy对象
    if (env->IsInstanceOf(obj, gBinderProxyOffsets.mClass)) {
        return (IBinder*)env->GetLongField(obj, gBinderProxyOffsets.mObject);
    }
    return NULL;
}
```

根据java创建的bbinder生成JavaBBinderHolder

### jbh->get(env, obj)

```java
sp<JavaBBinder> get(JNIEnv* env, jobject obj)
{
    AutoMutex _l(mLock);
    sp<JavaBBinder> b = mBinder.promote();
    if (b == NULL) {
        //首次进来，创建JavaBBinder对象【见3.5.4】
        b = new JavaBBinder(env, obj);
        mBinder = b;
    }
    return b;
}
```

data.writeStrongBinder(service)最终等价于`parcel->writeStrongBinder(new JavaBBinder(env, obj))`;

### parcel->writeStrongBinder

```java
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

### flatten_binder

```java
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    obj.flags = 0x7f | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    if (binder != NULL) {
        IBinder *local = binder->localBinder();
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.type = BINDER_TYPE_HANDLE; //远程Binder
            obj.binder = 0;
            obj.handle = handle;
            obj.cookie = 0;
        } else {
            obj.type = BINDER_TYPE_BINDER; //本地Binder，进入该分支
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        obj.type = BINDER_TYPE_BINDER;  //本地Binder
        obj.binder = 0;
        obj.cookie = 0;
    }
    //【见小节3.6.2】
    return finish_flatten_binder(binder, obj, out);
}
```

如果当前的bbinder是实体则创建binder_type_binder如果是引用则创建BINDER_TYPE_HANDLE

### **finish_flatten_binder**

```java
inline static status_t finish_flatten_binder(
    const sp<IBinder>& , const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```

最终将bbinder包装到flat结构体中。

### mRemote.transact

```java
public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    //用于检测Parcel大小是否大于800k
    Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
    return transactNative(code, data, reply, flags); //【见3.8】
}
```

### **android_os_BinderProxy_transact**

```java
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
    jint code, jobject dataObj, jobject replyObj, jint flags)
{
    ...
    //java Parcel转为native Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    ...

    //gBinderProxyOffsets.mObject中保存的是new BpBinder(0)对象
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //此处便是BpBinder::transact(), 经过native层，进入Binder驱动程序
    status_t err = target->transact(code, *data, reply, flags);
    ...
    return JNI_FALSE;
}
```

# 获取服务的过程

```java
public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name); //先从缓存中查看
        if (service != null) {
            return service;
        } else {
            return getIServiceManager().getService(name); 【见4.2】
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
```

getIServiceManager等价于servicemanagerproxy(new binderproxy()),而binderproxy = bpbinder（0）

```java
class ServiceManagerProxy implements IServiceManager {
    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        //mRemote为BinderProxy 【见4.3】
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        //从reply里面解析出获取的IBinder对象【见4.8】
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }
}
```

### binderproxy.transact

```java
final class BinderProxy implements IBinder {
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }
}
```

transactnative最终调用了android_os_BinderProxy_transact

```java
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
    jint code, jobject dataObj, jobject replyObj, jint flags)
{
    ...
    //java Parcel转为native Parcel
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);
    ...

    //gBinderProxyOffsets.mObject中保存的是new BpBinder(0)对象
    IBinder* target = (IBinder*)
        env->GetLongField(obj, gBinderProxyOffsets.mObject);
    ...

    //此处便是BpBinder::transact(), 经过native层[见小节4.5]
    status_t err = target->transact(code, *data, reply, flags);
    ...
    return JNI_FALSE;
}
```

其中gBinderProxyOffsets拿到bpbinder(0)的对象，最终调用transact将数据发送给binder启动。

```java
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        // [见小节4.6]
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```

### ipcthreadstate.transact

```java
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck(); //数据错误检查
    flags |= TF_ACCEPT_FDS;
    ....
    if (err == NO_ERROR) {
         // 传输数据
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    ...

    // 默认情况下,都是采用非oneway的方式, 也就是需要等待服务端的返回结果
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            //等待回应事件
            err = waitForResponse(reply);
        }else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```

### waitForResponse

```java
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ...
        cmd = mIn.readInt32();
        switch (cmd) {
          case BR_REPLY:
          {
            binder_transaction_data tr;
            err = mIn.read(&tr, sizeof(tr));
            if (reply) {
                if ((tr.flags & TF_STATUS_CODE) == 0) {
                    //当reply对象回收时，则会调用freeBuffer来回收内存
                    reply->ipcSetDataReference(
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t),
                        freeBuffer, this);
                } else {
                    ...
                }
            }
          }
          case :...
        }
    }
    ...
    return err;
}
```

### binder_send_reply

```java
void binder_send_reply(struct binder_state *bs, struct binder_io *reply, binder_uintptr_t buffer_to_free, int status) {
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;

    data.cmd_free = BC_FREE_BUFFER; //free buffer命令
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY; // reply命令
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        ...
    } else {=

        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    //向Binder驱动通信
    binder_write(bs, &data, sizeof(data));
}
```

将bc_reply写入结构体中，请求服务的进程执行talkWithDriver时执行到binder_thread_read，处理ToDo队列中的事物。

### readStrongBinder

```java
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr) {
    Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
    if (parcel != NULL) {
        //【见小节4.8.1】
        return javaObjectForIBinder(env, parcel->readStrongBinder());
    }
    return NULL;
}
```

将native层的bpbinder对象转换为Java层的binderproxy。

### parcel->readStrongBinder

```java
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    //【见小节4.8.2】
    unflatten_binder(ProcessState::self(), *this, &val);
    return val;
}
```

### unflatten_binder

```java
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat->type) {
            case BINDER_TYPE_BINDER:
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                //进入该分支【见4.8.3】
                *out = proc->getStrongProxyForHandle(flat->handle);
                //创建BpBinder对象
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

### getStrongProxyForHandle

```java
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    //查找handle对应的资源项
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            ...
            //当handle值所对应的IBinder不存在或弱引用无效时，则创建BpBinder对象
            b = new BpBinder(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```

<font color="red">最终将natvie层中指向服务端的bpbinder(handle)转换成了binderproxy对象公客户端调用。<font>