---
title: Fragment 中调用 requireContext() 报错 not attached to a context
date: 2021-12-19 00:13:31
tags:
- Android
- Bug
---

今天遇到个线上 Crash，报错信息是：`java.lang.IllegalStateException: Fragment ... not attached to a context.`。从堆栈初步分析，是调用 `requireContext` 导致的。考虑到之前也遇到过，但总是容易重新犯错，这次从源码角度分析下，加深理解。

<!-- more -->

## 前言

`requireContext` 是 AndroidX 中的类 `Fragment` 提供的 API，不属于 Android SDK。

Android SDK 中的类 `Fragment` 与之对应的方法是 `getContext`，这个方法 Android X 的类 `Fragment` 也有，并且作用一样。

细心的你会发现，在使用`getContext` 时，AS 会建议我们使用 `requireContext` 替代。

个人认为它这么做的原因是：`getContext` 返回的值**可能**为 `null`，但 `requireContext` 返回的值**一定不**为 `null`，使用后者的好处是避免了判空检查。

但如果你认为可以在 `Fragment` 内肆意地使用 `requireContext`，那就**大错特错**了。

在开始下一步分析前，先建立起一些共同的认知：

* `Fragment` 不能独立存在，必须依附 `Activity`。

## 源码分析

直接上源码：

```java
package androidx.fragment.app;

public class Fragment {
    /**
     * Return the {@link Context} this fragment is currently associated with.
     *
     * @see #requireContext()
     */
    @Nullable
    public Context getContext() {
        return mHost == null ? null : mHost.getContext();
    }

    /**
     * Return the {@link Context} this fragment is currently associated with.
     *
     * @throws IllegalStateException if not currently associated with a context.
     * @see #getContext()
     */
    @NonNull
    public final Context requireContext() {
        Context context = getContext();
        if (context == null) {
            throw new IllegalStateException("Fragment " + this + " not attached to a context.");
        }
        return context;
    }
}
```

可以看到，使用 `requireContext` 不需要进行空指针检查，是因为在它内部实现了判空逻辑。但这是以抛出异常为代价实现的，这告诫我们要慎用 `requireContext`。

可能有人会说，那我用 `requireContext` 之前进行次判空，或者直接用 `getContext` 不就行了。个人认为这并不是解决问题的根本方案。根本问题在于：我们调用 `requireContext` 的时候，是否“**应该**”出现 context 为 `null` 的情况，如果为 `null`，是不是认为此时程序执行的不合理了？是不是就应该 Crash？如果不应该为 `null`，是不是意味着我们的代码逻辑考虑不周，存在问题？从经验和事实来看，两种原因都有：前者多半是系统兼容性问题导致的，后者出现的次数会更多一些。

回到这次的问题，经排查发现，是由于在一个回调事件中，调用 `requireContext` 导致的异常。通过还原现场发现，用户其实已经退出了 Fragment（跟宿主 activity，或者说 context detach 掉），但是有一个回调在它之后触发，故而导致异常。

解决这个 Crash 的办法有很多，其中之一是在 `Fragment` 的 `onDetach` 中取消掉回调时间的注册，这是符合我们场景的预期的。

## `requireContext` 使用建议

* 在**回调中**调用 `requireContext ` 必须慎重！需要分析清楚当回调事件发生时，Fragment 可能存在的状态
  * 如果 Fragment 被 detach 掉后，访问 requireContext 的行为不符合预期，可以在 onDetach 事件中取消掉该回调的注册。
  * 如果确实需要，可以考虑使用其他 context 进行替代，比如 Application 的 context。
