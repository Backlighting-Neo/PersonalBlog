title: iOS中的ReactNative切换页面图片闪屏的问题
date: 2016-06-01 16:49:12
tags:
	- ReactNative
	- iOS
---

> 在原有的iOS项目中集成ReactNative后，ReactNative页面出现的时候，都会重新加载图片，会出现闪屏的现象，影响用户体验。原因在于每次ReactNative页面即将消失的时候，react会自动将所有的image对象置为空，释放掉所有图片的内存，这样内存会大幅度下降，当页面再次出现的时候，会重新init图片对象，导致闪屏。

<!-- more -->

---

原文链接 [崔大王的博客](http://cuidawang.github.io/) [iOS集成ReactNative后，切换页面图片闪屏](http://cuidawang.github.io/2016/06/01/iOS%E9%9B%86%E6%88%90ReactNative%E5%90%8E%EF%BC%8C%E5%88%87%E6%8D%A2%E9%A1%B5%E9%9D%A2%E5%9B%BE%E7%89%87%E9%97%AA%E5%B1%8F/)

# 问题背景

在原有的iOS项目中集成ReactNative后，ReactNative页面出现的时候，都会重新加载图片，会出现闪屏的现象，影响用户体验。
原因在于每次ReactNative页面即将消失的时候，react会自动将所有的image对象置为空，释放掉所有图片的内存，这样内存会大幅度下降，当页面再次出现的时候，会重新init图片对象，导致闪屏。

# 解决方案

```objectivec

- (void)clearImage
{
  [self cancelImageLoad];
  [self.layer removeAnimationForKey:@"contents"];
  //self.image = nil;
}

```

找到RCTImageView.m文件，将 - (void)clearImage 方法中的 self.image = nil; 注释掉，这样加载的图片在页面消失的时候就不会被释放了，就不会有闪屏的问题了。