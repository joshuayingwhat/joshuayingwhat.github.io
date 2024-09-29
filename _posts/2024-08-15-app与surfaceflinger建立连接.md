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

调用createconnection来请求surfaceflinger服务创建一个连接，创建一个类型为client的binder对象，并且将这个binder对象的代理接口conn返回。surfacecomposeclient类获得了surfaceflinger服务返回来的client代理接口conn后将其保存到自己的成员变量mClient中，这样应用程序就可以通过它来请求surfaceflinger创建和渲染surface了。

```java
Client::Client(const sp<SurfaceFlinger>& flinger)
    : mFlinger(flinger), mNameGenerator(1)
{
}
Client::~Client()
{
    const size_t count = mLayers.size();
    for (size_t i=0 ; i<count ; i++) {
        sp<LayerBaseClient> layer(mLayers.valueAt(i).promote());
        if (layer != 0) {
            mFlinger->removeLayer(layer);
        }
    }
}
```
Client类有两个成员变量mFlinger和mNameGenerator，它们的类型分别为sp<SurfaceFlinger>和int32_t，前者指向了SurfaceFlinger服务，而后者用来生成SurfaceFlinger服务为Android应用程序所创建的每一个Surface的名称。例如，假设一个Android应用程序请求SurfaceFlinger创建了两个Surface，那么第一个Surface的名称就由数字1来描述，而第二个Surface就由数字2来描述，依次类推。

![image.png](https://s2.loli.net/2024/09/29/8wiJLet4FvM6s1f.png)

这样app端通过SurfaceComposerClient与surfaceflinger建立了连接。
![](https://s2.loli.net/2024/08/15/zgq9PIHufZYwsSU.png)

<li>1. 应用创建SurfaceComposerClient对象，准备向SurfaceFlinger系统服务发起动作，由于SurfaceComposerClient继承自RefBase，因此会触发SurfaceComposerClient::onFirstRef函数。</li>
<li>2. 在onFirstRef函数中，SurfaceComposerClient通过IServiceManager::getService方法拿到了SurfaceFlinger系统的代理端BpSurfaceComposer。接口马上通过BpSurfaceComposer向SurfaceFlinger发起connect请求（Bp向Bn发起请求动作）。</li>
<ol>a. SurfaceFlinger接收到请求后，在本地创建一个本地代理对象Client（这是ClientBnSurfaceComposerClient端的），然后将该对象以ISrufaceComposerClient的形式返回给BpSurfaceComposer；</ol>
<ol>b. BpSurfaceComposer接收到SurfaceFlinger返回过来的Client对象后，通过ISurfaceComposerClient::asInterface函数将Client对象转换成BpSurfaceComposerClient对象。</ol>
<li>3. SurfaceComposerClient::onFirstRef函数执行完后，APP就与SurfaceFlinger服务成功建立连接了，后面就可以通过这个SurfaceFlinger的Client Proxy与SurfaceFlinger进行正常的通信。</li><br>

**<font color='red'>为什么有ISurfaceComposer还需要有Client?</font>**<br>
ISurfaceComposer是一个制定client和server的标准接口虽然也可以获取到surfaceflinger的实例，但是如果有多个实例就会有多个surfaceflinger不易维护资源管理混乱增加系统开销

而通过client的实例则可以让应用层创建多个client每个client又对应一个surface，因此一个应用可以有多个client对应的surface

参考:<br>
<https://www.jianshu.com/p/6964157c2615>

<https://blog.csdn.net/lijie2664989/article/details/115300781>

<https://www.cnblogs.com/tsts/p/9748042.html>

<https://blog.csdn.net/luoshengyang/article/details/7857163>
