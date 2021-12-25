---
title: View Binding 遇到 `java.lang.NullPointerException: Missing required view with ID`
date: 2021-12-25 21:31:25
tags:
- Android
---

今天使用 ViewBinding 时遇到一个 Crash：`java.lang.NullPointerException: Missing required view with ID`，最终发现是与**自定义 View** 有关系……

<!-- more -->

## 一、背景

最近使用 `ViewBinding` 时，遇到这么一个报错：

```bash
E AndroidRuntime: FATAL EXCEPTION: main
E AndroidRuntime: Process: me.hjhl.app, PID: 10740
E AndroidRuntime: java.lang.NullPointerException: Missing required view with ID: me.hjhl.app:id/my_gl_surface_view
E AndroidRuntime: 	at me.hjhl.app.databinding.FragmentGlesDemoBinding.bind(FragmentGlesDemoBinding.java:67)
E AndroidRuntime: 	at me.hjhl.app.databinding.FragmentGlesDemoBinding.inflate(FragmentGlesDemoBinding.java:49)
E AndroidRuntime: 	at me.hjhl.app.demo.GLESDemoFragment.onCreateView(GLESDemoFragment.kt:31)
E AndroidRuntime: 	at androidx.fragment.app.Fragment.performCreateView(Fragment.java:2995)
E AndroidRuntime: 	at androidx.fragment.app.FragmentStateManager.createView(FragmentStateManager.java:523)
E AndroidRuntime: 	at androidx.fragment.app.FragmentStateManager.moveToExpectedState(FragmentStateManager.java:261)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.executeOpsTogether(FragmentManager.java:1840)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.removeRedundantOperationsAndExecute(FragmentManager.java:1764)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.execPendingActions(FragmentManager.java:1701)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.dispatchStateChange(FragmentManager.java:2849)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.dispatchActivityCreated(FragmentManager.java:2784)
E AndroidRuntime: 	at androidx.fragment.app.FragmentController.dispatchActivityCreated(FragmentController.java:262)
E AndroidRuntime: 	at androidx.fragment.app.FragmentActivity.onStart(FragmentActivity.java:478)
E AndroidRuntime: 	at androidx.appcompat.app.AppCompatActivity.onStart(AppCompatActivity.java:246)
E AndroidRuntime: 	at android.app.Instrumentation.callActivityOnStart(Instrumentation.java:1433)
E AndroidRuntime: 	at android.app.Activity.performStart(Activity.java:7923)
E AndroidRuntime: 	at android.app.ActivityThread.handleStartActivity(ActivityThread.java:3337)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:221)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:201)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:173)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
E AndroidRuntime: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2049)
E AndroidRuntime: 	at android.os.Handler.dispatchMessage(Handler.java:107)
E AndroidRuntime: 	at android.os.Looper.loop(Looper.java:228)
E AndroidRuntime: 	at android.app.ActivityThread.main(ActivityThread.java:7589)
E AndroidRuntime: 	at java.lang.reflect.Method.invoke(Native Method)
E AndroidRuntime: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539)
E AndroidRuntime: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:953)
```

整体代码逻辑大致是：`Activity` 创建时跳转到一个 `Fragment`，这个 `Fragment` 对应的 XML 里很简单——`FrameLayout` 里套了一个自定义 View。错误堆栈提示找不到资源 id `my_gl_surface_view`。这让我不禁怀疑是不是自定义 View 导致的。

## 二、分析过程

### 2.1 找到 `ViewBinding` 生成的 UI 类

ViewBinding 的大致原理是在编译时，把开启了 ViewBinding 功能的布局资源 XML 生成对应的 Java 类。例如在这个 case 中，对应的类文件位置为：`app/build/generated/data_binding_base_class_source_out/debug/out/me/ljh/app/databinding/FragmentGlesDemoBinding.java`。

### 2.2 分析关键代码

以下代码片段是从 `FragmentGlesDemoBinding` 中摘出：

```java
// file: app/build/generated/data_binding_base_class_source_out/debug/out/me/ljh/app/databinding/FragmentGlesDemoBinding.java
public final class FragmentGlesDemoBinding implements ViewBinding {
  @NonNull
  public static FragmentGlesDemoBinding bind(@NonNull View rootView) {
  	// The body of this method is generated in a way you would not otherwise write.
    // This is done to optimize the compiled bytecode for size and performance.
    int id;
    missingId: {
    	id = R.id.my_gl_surface_view;
      MyGLSurfaceView myGlSurfaceView = ViewBindings.findChildViewById(rootView, id);
      if (myGlSurfaceView == null) {
        // 无法找到 view，则 break 准备抛出异常
        break missingId;
      }
      // 递归处理 R.id.my_gl_surface_view 的子 view
      return new FragmentGlesDemoBinding((FrameLayout) rootView, myGlSurfaceView);
    }
    String missingId = rootView.getResources().getResourceName(id);
    throw new NullPointerException("Missing required view with ID: ".concat(missingId));
  }
}
```

考虑到该问题跟布局文件密切相关，放上布局文件的源码：

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
  xmlns:tools="http://schemas.android.com/tools"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  android:background="?attr/fullscreenBackgroundColor"
  android:theme="@style/ThemeOverlay.LearnAndroidOpenGL.FullscreenContainer"
  tools:context=".demo.GLESDemoFragment">

  <!-- 这里使用了自定义布局 -->
  <me.ljh.app.widget.MyGLSurfaceView
    android:id="@+id/my_gl_surface_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

</FrameLayout>
```

其中，`MyGLSurfaceView` 的实现如下：

```kotlin
class MyGLSurfaceView(context: Context, attrs: AttributeSet?) : GLSurfaceView(context) {
 	// MyGLRender 是继承自 GLSurfaceView.Render 接口的一个类，与这里的报错并无关系
  private val renderer: MyGLRender = MyGLRender()

  init {
      setEGLContextClientVersion(3)
      setRenderer(renderer)
      renderMode = RENDERMODE_WHEN_DIRTY
  }

  constructor(context: Context) : this(context, null)
}
```

#### 2.2.1 怀疑点1：与 ViewBinding 不支持布局中有自定义的 View 控件有关？

虽然觉得不可能，但还是验证了下：

```kotlin
class GLESDemoFragment : Fragment() {
  
  private var _binding: FragmentGlesDemoBinding? = null
  
  private val mBinding get() = _binding!!
  
	override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
  ): View {
    // 步骤一：将 ViewBinding 的代码注释掉
    //_binding = FragmentGlesDemoBinding.inflate(inflater, container, false)
    // return mBinding.root
    // 步骤二：使用 inflate XML 的方式加载 View
    val rootView = inflater.inflate(R.layout.fragment_gles_demo, container, false)
    return rootView
  }
}
```

意外发现，**这样竟然不会奔溃，可以运行**了！

![App 成功运行](/images/app-run-success.png)

不会吧，真与 ViewBinding 有关？抱着怀疑、实事求是的态度，尝试在上面的基础上，通过 `findViewById` 拿到 view 实例。

```kotlin
val rootView = inflater.inflate(R.layout.fragment_gles_demo, container, false)
val myGlView = rootView.findViewById(R.id.my_gl_surface_view)
Log.d(TAG, "my gl view $myGlView")
return rootView
```

果不其然，竟然**获取到的为 `null`** ！！

这说明在根 view 中，找不到 ID 为 `my_gl_surface_view` 的子 view，但从这并不复杂的 XML 布局中，可以清晰看到，ID 确确实实是对的，哪里错了呢？并且，为什么通过手动 inflate 布局 XML 的方式，程序能运行的好好地呢？

### 2.3 发现问题

考虑到用了自定义 View，这种情况下，怀疑大概是自定义 View 写的有问题。

回到 `MyGLSurfaceView` 的源码，突然注意到它的**构造函数**。

```kotlin
class MyGLSurfaceView(context: Context, attrs: AttributeSet?) : GLSurfaceView(context) {
  // ......
  constructor(context: Context) : this(context, null)
}
```

最开始自定义 View 的时候，主构造函数只用到了一个 Context，然后发现在 XML 中引入布局后，运行程序会发生奔溃。

```bash
E AndroidRuntime: FATAL EXCEPTION: main
E AndroidRuntime: Process: me.ljh.app, PID: 16008
E AndroidRuntime: android.view.InflateException: Binary XML file line #13 in me.ljh.app:layout/fragment_gles_demo: Binary XML file line #13 in me.ljh.app:layout/fragment_gles_demo: Error inflating class me.ljh.app.widget.MyGLSurfaceView
E AndroidRuntime: Caused by: android.view.InflateException: Binary XML file line #13 in me.ljh.app:layout/fragment_gles_demo: Error inflating class me.ljh.app.widget.MyGLSurfaceView
E AndroidRuntime: Caused by: java.lang.NoSuchMethodException: me.ljh.app.widget.MyGLSurfaceView.<init> [class android.content.Context, interface android.util.AttributeSet]
E AndroidRuntime: 	at java.lang.Class.getConstructor0(Class.java:2332)
E AndroidRuntime: 	at java.lang.Class.getConstructor(Class.java:1728)
E AndroidRuntime: 	at android.view.LayoutInflater.createView(LayoutInflater.java:828)
E AndroidRuntime: 	at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:1010)
E AndroidRuntime: 	at android.view.LayoutInflater.createViewFromTag(LayoutInflater.java:965)
E AndroidRuntime: 	at android.view.LayoutInflater.rInflate(LayoutInflater.java:1127)
E AndroidRuntime: 	at android.view.LayoutInflater.rInflateChildren(LayoutInflater.java:1088)
E AndroidRuntime: 	at android.view.LayoutInflater.inflate(LayoutInflater.java:686)
E AndroidRuntime: 	at android.view.LayoutInflater.inflate(LayoutInflater.java:538)
E AndroidRuntime: 	at me.ljh.app.demo.GLESDemoFragment.onCreateView(GLESDemoFragment.kt:34)
E AndroidRuntime: 	at androidx.fragment.app.Fragment.performCreateView(Fragment.java:2995)
E AndroidRuntime: 	at androidx.fragment.app.FragmentStateManager.createView(FragmentStateManager.java:523)
E AndroidRuntime: 	at androidx.fragment.app.FragmentStateManager.moveToExpectedState(FragmentStateManager.java:261)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.executeOpsTogether(FragmentManager.java:1840)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.removeRedundantOperationsAndExecute(FragmentManager.java:1764)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.execPendingActions(FragmentManager.java:1701)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.dispatchStateChange(FragmentManager.java:2849)
E AndroidRuntime: 	at androidx.fragment.app.FragmentManager.dispatchActivityCreated(FragmentManager.java:2784)
E AndroidRuntime: 	at androidx.fragment.app.FragmentController.dispatchActivityCreated(FragmentController.java:262)
E AndroidRuntime: 	at androidx.fragment.app.FragmentActivity.onStart(FragmentActivity.java:478)
E AndroidRuntime: 	at androidx.appcompat.app.AppCompatActivity.onStart(AppCompatActivity.java:246)
E AndroidRuntime: 	at android.app.Instrumentation.callActivityOnStart(Instrumentation.java:1433)
E AndroidRuntime: 	at android.app.Activity.performStart(Activity.java:7923)
E AndroidRuntime: 	at android.app.ActivityThread.handleStartActivity(ActivityThread.java:3337)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:221)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:201)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:173)
E AndroidRuntime: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
E AndroidRuntime: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2049)
E AndroidRuntime: 	at android.os.Handler.dispatchMessage(Handler.java:107)
E AndroidRuntime: 	at android.os.Looper.loop(Looper.java:228)
E AndroidRuntime: 	at android.app.ActivityThread.main(ActivityThread.java:7589)
E AndroidRuntime: 	at java.lang.reflect.Method.invoke(Native Method)
E AndroidRuntime: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539)
E AndroidRuntime: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:953)
```

因此，给 `MyGlSurfaceView` 的主构造函数加入了 `AttributeSet` 参数，便没有了 Crash。但**并没有把该参数传递给父类** `GLSurfaceView`。

猜测很大概率是这里，因此加上后尝试下。

```kotlin
class MyGLSurfaceView(context: Context, attrs: AttributeSet?) : GLSurfaceView(context, attrs) {
  // ......
  constructor(context: Context) : this(context, null)
}
```

**通过 `findViewById` 能拿到 view 实例了，并且使用 ViewBinding 也正常了**。

看来真是这里的问题，但新问题来了，

1. 这个参数是干嘛用的？
2. 为什么 XML 方式创建 View 时一定需要它？
3. 为什么没有这个参数，findViewById 会找不到 View？如果确实要这样，有其他办法可以拿到吗？
4. 为什么自定义 View，通过 `setContenxtView(MyGLSurfaceView(this))` 类似的方式可以正常使用？

## 三、探究原理

### 3.1 `AttributeSet` 是什么？

在 AS 中，对着 `AttributeSet` 按 `Ctrl + B (Windows) / Command + B (MacOS)`，跳转到它的定义。

> ```java
> /**
>  * A collection of attributes, as found associated with a tag in an XML
>  * document.  Often you will not want to use this interface directly, instead
>  * passing it to {@link android.content.res.Resources.Theme#obtainStyledAttributes(AttributeSet, int[], int, int)
>  * Resources.Theme.obtainStyledAttributes()}
>  * which will take care of parsing the attributes for you.  In particular,
>  * the Resources API will convert resource references (attribute values such as
>  * "@string/my_label" in the original XML) to the desired type
>  * for you; if you use AttributeSet directly then you will need to manually
>  * check for resource references
>  * (with {@link #getAttributeResourceValue(int, int)}) and do the resource
>  * lookup yourself if needed.  Direct use of AttributeSet also prevents the
>  * application of themes and styles when retrieving attribute values.
>  * 
>  * <p>This interface provides an efficient mechanism for retrieving
>  * data from compiled XML files, which can be retrieved for a particular
>  * XmlPullParser through {@link Xml#asAttributeSet
>  * Xml.asAttributeSet()}.  Normally this will return an implementation
>  * of the interface that works on top of a generic XmlPullParser, however it
>  * is more useful in conjunction with compiled XML resources:
>  * 
>  * <pre>
>  * XmlPullParser parser = resources.getXml(myResource);
>  * AttributeSet attributes = Xml.asAttributeSet(parser);</pre>
>  * 
>  * <p>The implementation returned here, unlike using
>  * the implementation on top of a generic XmlPullParser,
>  * is highly optimized by retrieving pre-computed information that was
>  * generated by aapt when compiling your resources.  For example,
>  * the {@link #getAttributeFloatValue(int, float)} method returns a floating
>  * point number previous stored in the compiled resource instead of parsing
>  * at runtime the string originally in the XML file.
>  * 
>  * <p>This interface also provides additional information contained in the
>  * compiled XML resource that is not available in a normal XML file, such
>  * as {@link #getAttributeNameResource(int)} which returns the resource
>  * identifier associated with a particular XML attribute name.
>  *
>  * @see XmlPullParser
>  */
> ```

一大串有点长，总结就是：这是 XML 里面，对一个 View 配置的属性集合。比如`android:id="@+id/my_gl_surface_view"`、`android:layout_width="match_parent"`、`android:layout_height="match_parent"`，它们会被 XML Parser 解析成一个个 Java 对象，`AttributeSet` 就是这些对象的集合。

这就回答了前面的 1、2、3、4 个问题：

1. 这个参数，用于 XML 转换为 Java 对象时，传递 XML 中关于 view 实例的描述信息，包括它的`id`、宽、高……
2. 显然，通过 XML 绘制 View 的方式一定会有这个参数。
3. 没有这个参数，通过 `android:id="@+id/my_gl_surface_view` 方式指定的 id 自然就不会对 view 生效；如果一定要获取呢？既然能拿到父 view，就肯定有办法。我目前想到的是通过 `rootView.children` 的方式去遍历获取子 view 对象。但说实话没必要这样折腾自己。
4. 显然，通过代码的方式，当然可以不提供了。

## 四、结束语

感觉好快就破案了，貌似有些潦草……

其实这问题在于，**要理解 XML 到 UI 的原理**，如果理解了，那就不会很难。
