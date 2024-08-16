---
layout: post
title:  "四:APP与surfaceflinger服务连接过程"
categories: android
tags:  显示
---

* content
{:toc}

viewrootimpl内部会创建一个空的surface，还未被赋值
```java
public final Surface mSurface = new Surface();
```
程序执行完requestlayout之后最终会调用到session.addTodisplay,最终执行到wms@addWindow
```java
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, InputChannel outInputChannel) {
            win = new WindowState(this, session, client, token,
                    attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
            win.attach();
        return res;
    }

   WindowState@attach 
   void attach() {
        if (WindowManagerService.localLOGV) Slog.v(
            WindowManagerService.TAG, "Attaching " + this + " token=" + mToken
            + ", list=" + mToken.windows);
        mSession.windowAddedLocked();
    } 
    
    Session@windowAddedLocked
    void windowAddedLocked() {
        if (mSurfaceSession == null) {
            mSurfaceSession = new SurfaceSession();
            mService.mSessions.add(this);
        }
        mNumWindow++;
```
windowAddedLocked中创建了SurfaceSession对象。surfacesession的创建会调用nativeCreate进行jni调用

......

接着会执行到android_view_SurfaceSession.cpp中的nativeCreate().会创建SurfaceComposerClient对象
```java
static jlong nativeCreate(JNIEnv* env, jclass clazz) {
    SurfaceComposerClient* client = new SurfaceComposerClient();
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```
并将surfacecomposerclient的对象赋值给sp<SurfaceComposerClient>，client->incStrong((void*)nativeCreate);会导致onFirstRef函数会被调用
```java
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sm(getComposerService());
    if (sm != 0) {
        sp<ISurfaceComposerClient> conn = sm->createConnection();
        if (conn != 0) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}

sp<ISurfaceComposer> ComposerService::getComposerService() {
    return ComposerService::getInstance().mComposerService;
}
```
getComposerService就会调用getService获取到surfaceflinger服务的一个代理接口mComposerService
```java
ComposerService::ComposerService()
: Singleton<ComposerService>() {
    const String16 name("SurfaceFlinger");
    while (getService(name, &mComposerService) != NO_ERROR) {
        usleep(250000);
    }
    mServerCblkMemory = mComposerService->getCblk();
    mServerCblk = static_cast<surface_flinger_cblk_t volatile *>(
            mServerCblkMemory->getBase());
}
```
通过Surfaceflinger@getCblk获得一块匿名共享内存mServerCblkMemory，描述系统屏幕的宽高，个数，方向，密度等信息.

这个时候sm就是surfaceflinger的服务代理。SurfaceComposerClient有值后就会执行onFristRef函数.

![](https://s2.loli.net/2024/08/15/jbyuokKalAYFrXx.png)


这样app端通过SurfaceComposerClient与surfaceflinger建立了连接。
![](https://s2.loli.net/2024/08/15/zgq9PIHufZYwsSU.png)

<li>1. 应用创建SurfaceComposerClient对象，准备向SurfaceFlinger系统服务发起动作，由于SurfaceComposerClient继承自RefBase，因此会触发SurfaceComposerClient::onFirstRef函数。</li>
<li>2. 在onFirstRef函数中，SurfaceComposerClient通过IServiceManager::getService方法拿到了SurfaceFlinger系统的代理端BpSurfaceComposer。接口马上通过BpSurfaceComposer向SurfaceFlinger发起connect请求（Bp向Bn发起请求动作）。</li>
<ol>a. SurfaceFlinger接收到请求后，在本地创建一个本地代理对象Client（这是ClientBnSurfaceComposerClient端的），然后将该对象以ISrufaceComposerClient的形式返回给BpSurfaceComposer；</ol>
<ol>b. BpSurfaceComposer接收到SurfaceFlinger返回过来的Client对象后，通过ISurfaceComposerClient::asInterface函数将Client对象转换成BpSurfaceComposerClient对象。</ol>
<li>3. SurfaceComposerClient::onFirstRef函数执行完后，APP就与SurfaceFlinger服务成功建立连接了，后面就可以通过这个SurfaceFlinger的Client Proxy与SurfaceFlinger进行正常的通信。</li>


参考:
<https://www.jianshu.com/p/6964157c2615>

<https://blog.csdn.net/lijie2664989/article/details/115300781>

<https://www.cnblogs.com/tsts/p/9748042.html>

<https://blog.csdn.net/luoshengyang/article/details/7857163>
