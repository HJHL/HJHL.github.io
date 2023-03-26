---
title: Android Startup 使用和源码分析
date: 2023-03-26 20:04:04
tags:
- Android
- 源码分析
---

[Startup](https://developer.android.com/topic/libraries/app-startup)是 Jetpack 中用于解决应用程序启动时任务初始化的一个组件。

<!-- more -->

组件源码地址：[https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:startup/](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:startup/)

## 一、初见

从介绍看，它依靠 `ContentProvider` 完成了无侵入式组件初始化。所谓无侵入式，是指它不需要使用者通过手动方式调用进行初始化，而是依赖系统对四大组件之一 `ContentProvider` 的调用，间接对初始化内容进行调用。


看个使用例子。

通常项目中会有一个 Log 库，它一般不依赖其他任何业务组件，并且初始化的时机要尽可能前。

一般是这么初始化：
```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        // Log 库初始化
        Logger.init(this)
        // ...
    }
}
```

而通过 Startup 库，则是这么做：

1. 定义一个 `LoggerInitializer`，继承自 `Initializer`

```kotlin
package me.hjhl.app.demo

class LoggerInitializer : Initializer<Logger> {
    override fun create(context: Context): Logger {
        Logger.init(context)
        return Logger
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return emptyList();
    }
}
```

2. 在 `AndroidManifest.xml` 声明一个特定名称的 `ContentProvider`
```xml
<provider>
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <meta-data
        android:name="me.hjhl.app.demo.LoggerInitializer"
        android:value="androidx.startup">
</provider>
```

这样，就以 Startup 库的方式完成了 Log 组件的初始化。

乍一看，挺麻烦的，比前一种方式的代码要多得多，还引入了一个 `ContentProvider`，会带来性能上的损失。那它的优点在哪里？

## 二、优点 & 缺点
项目中接过新组件的同学应该遇到过：组件 A 初始化依赖组件 B 提供信息，因此组件 A 必须在组件 B 初始化完成后，才能进行初始化工作。在中大型项目中，组件多，这种依赖关系会很复杂，一不留神就容易引起错误。

Startup 就是给我们解决这个问题的——它可以初始化组件，并完成该组件所依赖组件的初始化。

假设我们有一个 AB 实验组件，一个网络组件。显然，AB 组件依赖网络组件完成后，才能正常使用其功能。

借助 Startup，可以这么实现：
```kotlin
package me.hjhl.app.demo

class NetworkInitializer : Initializer<NetworkManager> {
    override fun create(context: Context): NetworkManager {
        val configuration = NetworkInitParamsBuilder
                                .setUrl("https://xxx.xxx.xxx")
                                .build()
        NetworkManager.init(configuration)
        return NetworkManager
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        // 网络组件依赖 Log 组件
        return listOf(Logger::class.java)
    }
}
```
```kotlin
package me.hjhl.app.demo

class AbTestInitializer : Initializer<AbTestManager> {
    override fun create(context: Context): AbTestManager {
        val configuration = AbTestManagerInitParamsBuilder
                                .setUrl("https://xxx.xxx.xxx")
                                .setNetwork { url ->
                                    return NetworkManager.postSync(url)
                                }
                                .build()
        AbTestManager.init(configuration)
        return AbTestManager
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(NetworkInitializer::class.java)
    }
}
```
```xml
<provider>
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <meta-data
        android:name="me.hjhl.app.demo.AbTestInitializer"
        android:value="androidx.startup">
</provider>
```

Startup 的缺点也很明显：除了新增一个 `ContentProvider` 增加耗时外，还要求组件初始化必须同步（或者通过异步转同步）完成。

## 三、源码分析
首先，看下新增的 `ContentProvider`。

```java
// file: startup/startup-runtime/src/main/java/androidx/startup/InitializationProvider.java
public class InitializationProvider extends ContentProvider {
    @Override
    public final boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            Context = applicationContext = context.getApplicationContext();
            if (applicationContext != null) {
                AppInitializer.getInstance(context).discoverAndInitialize(getClass());
            } else {
                StartupLogger.w("Deferring initialization because `applicationContext` is null.");
            }
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
    // ...
}
```
这里关键是调用 `AppInitializer`，完成后续的整个初始化链。

那来看下 `AppInitializer`。
```kotlin
// file: startup/startup-runtime/src/main/java/androidx/startup/AppInitializer.java
public final class AppInitializer {
    /**
     * 单例模式，延迟初始化
     */
    public static AppInitializer getInstance(@NonNull Context context) {
        if (sInstance == null) {
            synchronized (sLock) {
                if (sInstance == null) {
                    sInstance = new AppInitializer(context);
                }
            }
        }
        return sInstance;
    }

    void discoverAndInitialize(
        @NonNull Class<? extends InitializationProvider> initializationProvider
    ) {
        try {
            Trace.beginSection(SECTION_NAME);
            // 获取 provider 的 meta data
            ComponentName provider = new ComponentName(mContext, initializationProvider);
            ProviderInfo providerInfo = mContext.getPackageManager()
                    .getProviderInfo(provider, GET_META_DATA);
            Bundle metadata = providerInfo.metaData;
            // 进行真正的初始化
            discoverAndInitialize(metadata);
        } catch (PackageManager.NameNotFoundException exception) {
            throw new StartupException(exception);
        } finally {
            Trace.endSection();
        }
    }

    void discoverAndInitialize(@Nullable Bundle metadata) {
        String startup = mContext.getString(R.string.androidx_startup);
        try {
            if (metadata != null) {
                Set<Class<?>> initializing = new HashSet<>();
                Set<String> keys = metadata.keySet();
                // 遍历所有 metadata，注意：key 是各个 Initializer 类的全名
                for (String key : keys) {
                    // 只有 value 满足的 Initializer 才进行初始化
                    String value = metadata.getString(key, null);
                    if (startup.equals(value)) {
                        Class<?> clazz = Class.forName(key);
                        // 检查 key 所表示的类是不是继承自 Initializer
                        if (Initializer.class.isAssignableFrom(clazz)) {
                            Class<? extends Initializer<?>> component =
                                    (Class<? extends Initializer<?>>) clazz;
                            // 满足所有条件，添加到已发现的列表中
                            mDiscovered.add(component);
                            if (StartupLogger.DEBUG) {
                                StartupLogger.i(String.format("Discovered %s", key));
                            }
                        }
                    }
                }
                // Initialize only after discovery is complete. This way, the check for
                // isEagerlyInitialized is correct.
                // meta data 中的 Initializer 发现完毕，开始对齐初始化
                for (Class<? extends Initializer<?>> component : mDiscovered) {
                    doInitialize(component, initializing);
                }
            }
        } catch (ClassNotFoundException exception) {
            throw new StartupException(exception);
        }
    }

    private <T> T doInitialize(
            @NonNull Class<? extends Initializer<?>> component,
            @NonNull Set<Class<?>> initializing) {
        boolean isTracingEnabled = Trace.isEnabled();
        try {
            if (isTracingEnabled) {
                // Use the simpleName here because section names would get too big otherwise.
                Trace.beginSection(component.getSimpleName());
            }
            // 检查之前是否初始化过，注意：是指从本次调用最开始的时刻开始
            if (initializing.contains(component)) {
                String message = String.format(
                        "Cannot initialize %s. Cycle detected.", component.getName()
                );
                throw new IllegalStateException(message);
            }
            Object result;
            if (!mInitialized.containsKey(component)) {
                initializing.add(component);
                try {
                    // 获取 Initializer 实例
                    Object instance = component.getDeclaredConstructor().newInstance();
                    Initializer<?> initializer = (Initializer<?>) instance;
                    // 收集依赖信息
                    List<Class<? extends Initializer<?>>> dependencies =
                            initializer.dependencies();

                    // 如果存在依赖，遍历依赖，检查是否初始化过（进程周期内）
                    // 如果依赖没有初始化，则递归调用完成依赖的初始化
                    if (!dependencies.isEmpty()) {
                        for (Class<? extends Initializer<?>> clazz : dependencies) {
                            if (!mInitialized.containsKey(clazz)) {
                                doInitialize(clazz, initializing);
                            }
                        }
                    }
                    if (StartupLogger.DEBUG) {
                        StartupLogger.i(String.format("Initializing %s", component.getName()));
                    }
                    // 这里，所有依赖都满足了，可以对 Initializer 进行初始化
                    result = initializer.create(mContext);
                    if (StartupLogger.DEBUG) {
                        StartupLogger.i(String.format("Initialized %s", component.getName()));
                    }
                    initializing.remove(component);
                    mInitialized.put(component, result);
                } catch (Throwable throwable) {
                    throw new StartupException(throwable);
                }
            } else {
                result = mInitialized.get(component);
            }
            return (T) result;
        } finally {
            Trace.endSection();
        }
    }
}
```

## 四、感想
Q：如果希望使用这套依赖发现机制，又不想引入 `ContentProvidr`，该如何办？
A：简单，将 `provider` 标记 `tools:node="remove"`，然后在代码恰当的位置，调用 `AppInitializer.getInstance(context).initializeComponent(YourInitializer::class.java)`。