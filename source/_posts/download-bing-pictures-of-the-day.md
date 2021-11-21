---
title: 「Kotlin 101」下载必应每日一图
date: 2021-11-21 21:07:55
tags:
- Kotlin
- Toys
---

记得之前在学 Python 爬虫的时候，那必应的每日一图练过手。这次看看如何**用 Kotlin 实现必应每日一图的下载**。

> 本文的主要目的，是为了熟悉 Kotlin 使用。

<!-- more -->

## 思路

把大象放进冰箱要几步？……啊，不对，此处应是“把网络上的一个图片下载下来需要几步”？



真的很“简单”：

1. 获取图片下载链接
2. 下载图片
3. 保存图片到文件



跟“大象放冰箱”一样，也是三步，但其实理解每一步的知识点，就不难。

## 步骤

### 1. 获取图片下载链接

> 这里涉及到一个知识点：**Json 反序列化**。

以前为了获取图片的下载链接，需要解析文档，但现在 Bing 提供了一个 API 直接获取图片的下载链接。

```bash
https://cn.bing.com/hp/api/model
```

这个 API 返回的是 Json 格式的字符串，里面包含了图片的下载地址。

![浏览器访问，看下 API 返回结果](/images/bing-api-result.png)

这里返回了一堆，不太好看，可以找个在线格式化 Json 字符串的网站看下。

![在线格式化 Json 字符串结果](/images/bing-pics-download-url.png)

这里 `Url` 对应的值即为图片下载地址的 `Path`。当然，它并不完整，需要加上域名后才是完整的下载链接。

```bash
https://cn.bing.com/th?id=OHR.Invergarry_EN-CN6569359503_1920x1080.jpg&rf=LaDigue_1920x1080.jpg
```

![2021-11-21 每日一图](https://cn.bing.com/th?id=OHR.Invergarry_EN-CN6569359503_1920x1080.jpg&rf=LaDigue_1920x1080.jpg)



那么，回到主题，在 Kotlin 代码中如何解析 Json 呢？

可能有的小伙伴会想到 `Gson`，确实，这是一个非常强大的序列化/反序列化的三方库，它也支持 Kotlin。但今天，这里使用 Kotlin 官方的一个序列化/反序列化库：`kotlin serialization`。

#### 引入`Kotlin serialization`

我这里使用的了 Kotlin DSL 作为构建语言，因此需要在 `build.gradle.kts` 中，添加如下字段：

```kotlin
plugins {
  ...
  kotlin("plugin.serialization") version "1.6.0"
}

dependencies {
  ...
  implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.3.1")
}
```

如果你使用的是 `Groovy`，请在 `build.gradle` 中完成修改。

#### 定义数据结构

`kotlin serialization `支持两种操作：

* 将一个对象序列化为 Json 字符串
* 将 Json 字符串反序列化成一个对象

这一切是有前提的，就是我们必须定义，并通过特定注解修饰这个对象的数据结构。

这里使用到了 `@Serializable`，目的是告诉 `Kotlin serialization` 如何序列化/反序列化该对象。

```kotlin
@Serializable
data class RequestJson(val MediaContents: List<MediaContent>) {

    @Serializable
    data class MediaContent(val ImageContent: ImageContent)

    @Serializable
    data class ImageContent(val Image: Image, val Title: String)

    @Serializable
    data class Image(val Url: String, val Wallpaper: String)
}
```

> 建议对照 Json 格式化后的结果看上述代码。

#### 反序列化生成对象

```kotlin
private val json = Json { ignoreUnknownKeys = true }
// 假设 imageInfo 为通过上述 API 获取到的 Request Body
val requestJson = json.decodeFromString<RequestJson>(imageInfos)
```

这里有一个坑，注意在构建 Json 对象（用于序列化和反序列化）时，`ignoreUnknownKeys` 必须设为 `true`，因为我们并没有把返回的所有 Json 元素都反序列化，只时挑出了我们需要的部分。而 `kotlin serialization` 是默认需要反序列化所有元素的，因此这里需要显示指明。

### 2. 下载图片

由于要从网上访问资源，所以这里需要熟悉 Kotlin 中网络接口相关的API/三方库。

三方库可以选用 `OkHttp` 或者 `Ktor`，但考虑到 Kotlin 是完全兼容 Java 的，所以还可以使用 JDK 中的 `URL` 和`HttpClient`/`HttpRequest`。

```kotlin
/**
 * 下载文件
 */
private fun download(url: String, path: String) {
    URL(url).openStream().use { input ->
        FileOutputStream(File(path)).use { output ->
            input.copyTo(output)
        }
    }
}

/**
 * 获取图片下载链接信息
 */
private suspend fun getImageInfo() = suspendCancellableCoroutine<String> { cont ->
    try {
        val client = HttpClient.newBuilder().build()
        val request = HttpRequest.newBuilder().uri(URI.create(API)).build()
        val response = client.send(request, HttpResponse.BodyHandlers.ofString())
        cont.resume(response.body())
    } catch (e: Exception) {
        cont.resumeWithException(e)
    }
}
```

### 3. 将图片保存到文件

第2步中已经实现了。



## 源码

[HJHL/KotlinToys](https://github.com/HJHL/KotlinToys/blob/main/src/main/kotlin/BingPicsDownload.kt)
