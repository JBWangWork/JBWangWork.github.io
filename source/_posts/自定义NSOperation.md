---
title: 自定义NSOperation子类-图片下载器
date: 2020-01-07 18:37:57
author: Vincent
categories: 
- 底层原理
- 源码分析
tags: 
- NSOperation
- 源码分析
- 底层原理
---


![自定义NSOperation子类-图片下载器](https://upload-images.jianshu.io/upload_images/5741330-a8bb9a75deb0f182.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 研究过NSOperation后，想通过实战更好的理解NSOperation，适用于对下载图片不频繁的项目，免得为了一个小需求而导入比较重的框架。[Demo(直通车)](https://github.com/JBWangWork/VWebImage)主要利用自定义NSOperation子类，同时借鉴了`AFNetworking、SDWebImage、YYKit`的部分思想来实现具有缓存支持的异步图像下载器。

#### 架构思想

整个架构共分为三层：Manager（单例，主要来管理总体的控制）、Operation（进行队列的控制）和UIImageView（对外）层。通过UIImageView的分类来通知Manager，Manager中注册了清理操作通知，Manager接收到信号后再把信号下发到对应的Operation中，Operation进行队列的操作。

UIImageView的分类主要对外提供接口，内部通过URL进行判断当前操作是否需要下载，如果需要下载则通知Manager来进行下载，下载完成后设置image。UIImageView分类的部分代码：
```
- (void)v_setImageWithUrlString:(NSString *)urlString title:(NSString *)title {
    if (!urlString) {
        VLog(@"下载地址为空");
        return;
    }
    
    if ([self.v_urlString isEqualToString:urlString]) {
        VLog(@"%@ 两次下载地址一样的 没必要重复下载",title);
        return;
    }
    
    if (self.v_urlString && self.v_urlString.length > 0 && ![self.v_urlString isEqualToString:urlString]) {
        VLog(@"取消之前的下载操作:%@---%@ \n%@---%@",self.v_title,title,self.v_urlString,urlString);
        [[VWebImageManager sharedManager] cancelDownloadImageWithUrlString:self.v_urlString];
    }
    //记录新操作开始下载
    self.v_urlString = urlString;
    self.v_title = title;
    // 复用
    self.image = nil;
    
    [[VWebImageManager sharedManager] downloadImageWithUrlString:urlString completeHandle:^(UIImage *downloadImage,NSString *urlString) {
        //下载完成 要置空
        if ([urlString isEqualToString:self.v_urlString]) {
            self.v_urlString = @"";
            self.v_title = @"";
            self.image = downloadImage;
        }
    } title:title];
}

- (void)v_setImageWithUrlString:(NSString *)urlString {
    [self v_setImageWithUrlString:urlString title:@""];
}
```

Manager接收到需要下载（或者取消下载）的信号后，首先从内存获取图片，有则利用block直接返回，否则继续从沙盒获取图片（内存的读取速度比沙盒快，所以先读内存），找到了则返回并把图片存入缓存。如果还是没有找到则进行判断是否需要下载，如果当前URL已经在下载中，则不需要再次进行下载，否则通知Operation来进行创建下载操作。Manager部分代码：
```
- (void)downloadImageWithUrlString:(NSString *)urlString completeHandle:(VCompleteHandle)completeHandle title:(NSString *)title {
    //内存获取图片
    UIImage *cacheImage = self.imageCacheDict[urlString];
    if (cacheImage) {
        VLog(@"从内存缓存获取数据 %@",title);
        completeHandle(cacheImage,urlString);
        return;
    }
    //沙盒获取图片
    NSString *cachePath = [urlString getDowloadImagePath];
    cacheImage = [UIImage imageWithContentsOfFile:cachePath];
    if (cacheImage) {
        VLog(@"从沙盒缓存获取数据 %@",title);
        //沙盒图片 存入缓存
        [self.imageCacheDict setObject:cacheImage forKey:urlString];
        completeHandle(cacheImage,urlString);
        return ;
    }
    //对当前下载图片判断,是否需要创建操作
    if (self.operationDict[urlString]) {
        VLog(@"正在下载的回调Block %@的%@",title,completeHandle);
        NSMutableArray *mArray = self.handleDict[urlString];
        if (mArray == nil) {
            mArray = [NSMutableArray arrayWithCapacity:1];
        }
        [mArray addObject:completeHandle];
        [self.handleDict setObject:mArray forKey:urlString];
        return;
    }
    // 创建操作 下载 --- 自定义
    VWebImageDownloadOperation *downOp = [[VWebImageDownloadOperation alloc] initWithDownloadImageUrl:urlString completeHandle:^(NSData * _Nonnull imageData, NSString * _Nonnull v_urlString) {
        UIImage *downloadImage = [UIImage imageWithData:imageData];
        if (downloadImage) {
            [self.imageCacheDict setObject:downloadImage forKey:urlString];
            [self.operationDict removeObjectForKey:urlString];
            [[NSOperationQueue mainQueue] addOperationWithBlock:^{
                completeHandle(downloadImage, v_urlString);
                // 去取回调
                if (self.handleDict[v_urlString]) {
                    NSMutableArray *mArray = self.handleDict[v_urlString];
                    for (VCompleteHandle completeHandle in mArray) {
                        completeHandle(downloadImage, v_urlString);
                    }
                    [self.handleDict removeObjectForKey:urlString];
                }
            }];
        }
    } title:title];
    // 操作加入队列
    [self.queue addOperation:downOp];
    // 操作缓存
    [self.operationDict setObject:downOp forKey:urlString];
}
```

自定义NSOperation子类需要重写Start等函数，这里参考了SDWebImage的思想进行手动KVO观察当前的状态，防止自动KVO的影响。比如这里可以根据相应的url进行取消当前任务，而此时GCD就无法满足需求。自定义NSOperation子类部分代码：
```
- (instancetype)initWithDownloadImageUrl:(NSString *)urlString completeHandle:(VDownCompleteHandle)completeHandle title:(NSString *)title{
    if (self = [super init]) {
        _urlString = urlString;
        _executing = NO;
        _finished  = NO;
        _cancelled = NO;
        _lock      = [NSRecursiveLock new];
        _completeHandle = completeHandle;
        _title = title;
    }
    return self;
}
```


#### 使用
```
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    VCustomCollectionViewCell *cell  = [collectionView dequeueReusableCellWithReuseIdentifier:reuseID forIndexPath:indexPath];
    VModel *model        = self.dataArray[indexPath.row];
    cell.titleLabel.text  = model.title;
    cell.moneyLabel.text  = model.money;
    [cell.imageView v_setImageWithUrlString:model.imageUrl];
    // debug
    // [cell.imageView v_setImageWithUrlString:model.imageUrl title:model.title];
    return cell;
}
```
Demo地址：[https://github.com/JBWangWork/VWebImage](https://github.com/JBWangWork/VWebImage)

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

