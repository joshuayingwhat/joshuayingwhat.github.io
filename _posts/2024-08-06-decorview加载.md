---
layout: post
title:  "二:decorview加载"
categories: android
tags:  显示
---

* content
{:toc}

当系统调用启动一个activity是就会通过反射启动,然后调用handlelauncheractivity。在handlelauncheracitiviy中执行mInstrumentation.callActivityOnCreate调用activity的oncrreate，并且调用activity.attach函数创建窗口视图phonewindow。
```java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
                activity.attach(activityBaseContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                        r.assistToken, r.shareableActivityToken, r.initialCallerInfoAccessToken);
                r.activity = activity;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            }
            r.setState(ON_CREATE);
       
        return activity;
    }
```

......

activity中oncreate的setcontentview函数就会被调用这个时候系统就会初始化decorview并且解析xml
```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
    }
```
其中getwidnow其实就是activiy.attach中创建的phonewindow。
```java
public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else {
            mContentParent.removeAllViews();
        }
        mLayoutInflater.inflate(layoutResID, mContentParent);
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
    }
```
mContentParent就是app中的xml布局的父布局，在installdecor中初始化出来。installdecor主要做两件事一个是创建decorview第二个就是找到decorview中的容器mContentparent
```java
private void installDecor() {
        if (mDecor == null) {
            mDecor = generateDecor();
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
        }
    }

protected DecorView generateDecor() {
        return new DecorView(getContext(), -1);
    }
```
generateDecor就是new Decorview，创建decorview对象。generateLayout将创建出来的mDecorview作为参数.
```java
protected ViewGroup generateLayout(DecorView decor) {
        mDecor.startChanging();
        View in = mLayoutInflater.inflate(layoutResource, null);
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
        if (contentParent == null) {
            throw new RuntimeException("Window couldn't find content container view");
        }
        mDecor.finishChanging();
        return contentParent;
    }
```
其中ID_ANDROID_CONTENT就是decorview中的一个布局id。接下来解析布局mLayoutInflater.inflate(layoutResID, mContentParent);
```java
 public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            View result = root;
            if (root != null && root.getViewRootImpl() != null) {
                root.getViewRootImpl().notifyRendererOfExpensiveFrame();
            }
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }
            return result;
        }
    }
```
root就是传进来的mCotnentParent。最终将我们的xml布局添加到decorview的布局中。最终decorview已经有布局内容了,接下来执行handleresumactiviy中的makevisible函数
```java
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```
最总调用windowmanager的addview方法。将我们的decorview添加到创建出来的phonewindow中。

参考:[https://www.jianshu.com/p/c2b38bada5ba]