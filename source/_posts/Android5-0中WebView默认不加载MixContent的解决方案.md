title: Android5.0中WebView默认不加载Mixed Content的解决方案
tags:
  - https
  - Mixed Content
  - WebView
  - Upgrade Insecure Requests
date: 2016-02-26 16:07:27
---
>  前文 ( [HTTPS中证书链不完整的解决方案](/2016/02/24/HTTPS中证书链不完整的解决方案/) ) 中提到我们的业务是在原生APP中嵌入WebView来做的。但在更换了HTTPS协议之后，却发现在一部分安卓手机上，页面上的Mixed Content 无法加载……

> 这里的Mixed Content 指 Image, Video 等 Optionally-blockable 类的资源，非 js, css 类的Blockable资源

<!-- more -->

---

# 问题背景

前文 ( [HTTPS中证书链不完整的解决方案](/2016/02/24/HTTPS中证书链不完整的解决方案/) ) 中提到我们的业务是在原生APP中嵌入WebView来做的。但在更换了HTTPS协议之后，却发现在一部分安卓手机上，页面上的Mixed Content 无法加载。

后来发现所有无法正常加载图片类 Mixed Content 资源的手机都是 Android 5.0 系统及以上的系统中自带的 WebView ，但不是全部，在 Android 5.1 系统的 魅族MX4 ( Flyme 5.6.2.23 ) 手机上可以加载Mixed Content

# Mixed Content

先从 Mixed Content 说起，这个概念是伴随着 https 协议出现而出现的。

HTTPS 网页中加载的 HTTP 资源被称之为 Mixed Content（混合内容），不同浏览器对 Mixed Content 有不一样的处理规则。

HTTPS 网页中加载的 HTTP 资源被称之为 Mixed Content（混合内容），不同浏览器对 Mixed Content 有不一样的处理规则。

现代浏览器（Chrome、Firefox、Safari、Microsoft Edge），基本上都遵守了 W3C 的 Mixed Content 规范，将 Mixed Content 分为 Optionally-blockable 和 Blockable 两类：

Optionally-blockable 类 Mixed Content 包含那些危险较小，即使被中间人篡改也无大碍的资源。现代浏览器默认会加载这类资源，同时会在控制台打印警告信息。这类资源包括：

- 通过 img 标签加载的图片（包括 SVG 图片）；
- 通过 video / audio 和 source 标签加载的视频或音频；
- 预读的（Prefetched）资源；

除此之外所有的 Mixed Content 都是 Blockable，浏览器必须禁止加载这类资源。所以现代浏览器中，对于 HTTPS 页面中的 JavaScript、CSS 等 HTTP 资源，一律不加载，直接在控制台打印错误信息。

# Android 5.0中WebView的行为

纠结了很长时间之后，用Chrome的远程调试发现了问题，在 Android 5.0 之后的WebView中是默认不加载任何类型的Mixed Content的资源。

要解决这个问题，无非两种途径，一种是要求原生开发的同学给我们对APP中的WebView做相关的调整，要么我们替换页面中的所有资源为 https 的链接。

# 解决方案

## 修改WebView行为

下面是Andorid Developers中的介绍

> **public abstract void setMixedContentMode (int mode)**

> Configures the WebView's behavior when a secure origin attempts to load a resource from an insecure origin. By default, apps that target KITKAT or below default to MIXED_CONTENT_ALWAYS_ALLOW. Apps targeting LOLLIPOP default to MIXED_CONTENT_NEVER_ALLOW. The preferred and most secure mode of operation for the WebView is MIXED_CONTENT_NEVER_ALLOW and use of MIXED_CONTENT_ALWAYS_ALLOW is strongly discouraged.

> **mode** *int* 

> The mixed content mode to use. One of MIXED_CONTENT_NEVER_ALLOW, MIXED_CONTENT_ALWAYS_ALLOW or MIXED_CONTENT_COMPATIBILITY_MODE.

[https://developer.android.com/reference/android/webkit/WebSettings.html#MIXED_CONTENT_NEVER_ALLOW](https://developer.android.com/reference/android/webkit/WebSettings.html#MIXED_CONTENT_NEVER_ALLOW)


一句话来说，在Android5.0之前，KITKAT 以及之前的Android中的WebView是默认加载图片类的Mixed Content资源的，从 5.0 LOLLIPOP 之后默认不加载任何 Mixed Content 资源。

要修改这种默认的行为，可以给 WebView 做这样的设置

```Java
webView.getSettings().setMixedContentMode(WebSettings.MIXED_CONTENT_COMPATIBILITY_MODE);
```

这样是允许图片类的 Mixed Content 资源的加载，要加载全部的 Mixed Content 资源，包括js等，可以使用下边的设置 （这样做很危险）

```Java
webView.getSettings().setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);
```

需要注意的是，这两个API需要在Android API 21 以上才能运行，执行前需要加SDK版本的判断，因为 KITKAT 之前，默认就是可以加载不安全资源的，所以可以放心使用。

## Upgrade Insecure Requests

### upgrade-insecure-requests CSP指令

历史悠久的大站在往 HTTPS 迁移的过程中，工作量往往非常巨大，尤其是将所有资源都替换为 HTTPS 这一步，很容易产生疏漏。即使所有代码都确认没有问题，很可能某些从数据库读取的字段中还存在 HTTP 链接。

而通过 upgrade-insecure-requests 这个 CSP 指令，可以让浏览器帮忙做这个转换。启用这个策略后，有两个变化：

- 页面所有 HTTP 资源，会被替换为 HTTPS 地址再发起请求；
- 页面所有站内链接，点击后会被替换为 HTTPS 地址再跳转；

跟其它所有 CSP 规则一样，这个指令也有两种方式来启用。需要注意的是 upgrade-insecure-requests 只替换协议部分，所以只适用于 HTTP/HTTPS 域名和路径完全一致的场景。

### 启用方式

要启用CSP指令，有两种方式，如果你的服务器在自己的控制下，可以自由的输出HTTP头的话，你可以在HTTP头中加入

```
Content-Security-Policy: upgrade-insecure-requests
```

如果你的服务器是虚拟主机，或是像七牛的云存储，总之不能自己输出HTTP头，可以在 Html 文件的 head 区中加入 **meta** 标签

```html
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
```

### 影响

加入了 **Upgrade Insecure Requests** 指令后，页面中的所有 **http请求** 都会被浏览器在 fetch 的时候，转变成 **https请求**，所有的页面跳转，包括 **location.href** 也会被转换后再发起 ，注意是所有请求，如果你某些资源域名上没有开启https，将会导致加载失败。