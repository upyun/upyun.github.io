---
layout: post
title:  "ALAsset/PHAsset 中的图片和视频文件"
author: rotoava
comments: true
---

> 在微博上出现了越来越多的被标记为 Live 的图片，这种图片是一种动图 LivePhoto，长按之后会进行播放。那么 LivePhoto 是一种什么文件或者格式？其实 LivePhoto 只是一种 iPhone 相册的资源 Asset，并不是一种特殊的动图文件和格式。下面将通过对 PHAsset 的使用过程来描述视频 Asset，图片 Asset 和 LivePhoto Asset 与其真正包含文件的关系。



## 1.关键词

__ALAsset, PHAsset, Photos library, UIImagePickerController, PHLivePhoto, LivePhoto__

ALAsset 或者 PHAsset 代表着由 iPhone 相册 app 管理的视频和图片对象。ALAsset 在 iOS9.0 版本已经被弃用，PHAsset 是 ALAsset 的替代。和手机相册（Photos）进行的交互，比如选择图片上传，都会涉及到 ALAsset/PHAsset 相关的概念。

``` 
//使用 ALAsset 需要引入 AssetsLibrary。 AssetsLibrary 在 iOS9.0 已经过期。
#import <AssetsLibrary/AssetsLibrary.h>
//使用 PHAsset 需要引入Photos Framework，支持 iOS8.0 及以上版本。
#import <Photos/Photos.h> 
```

ALAsset/PHAsset 并不是真正的文件对象，他们仅仅包含真正文件的基本信息如：文件路径，文件元数据。甚至一个 Asset 会包含多个文件 (多个 ALAssetRepresentation 或者 PHAssetResource), 如 _LivePhoto_ 包含一个 jpeg 图片和一个 mov 视频两个文件。     

LivePhoto 是在 iPhone6s 及更新的设备上用相机拍摄的一张照片，其特点是包含了照片拍摄时刻之前和之后几秒钟的视频（拍摄 LivePhoto 需要 iPhone6s 及更新的设备；LivePhoto 的操作和播放只需要安装了 iOS9.1 及以上系统版本的 iPhone 即可），LivePhoto 不是一种新文件格式，LivePhoto 只是一种特别的 PHAsset。

ALAsset/ PHAsset 对象较为复杂，所以理清 ALAsset/PHAsset 和真正文件的关系，才能使后续的视频和图片文件的操作，比如上传CDN，变的好理解。

__下面以一个常见的使用场景进行 PHAsset 操作过程的描述：__

_从相册选择图片或视频 — 将图片或视频上传CDN — 下载图片或视频 — 将图片或视频保存到相册_ 

(ALAsset 已在 iOS9.0 过期，所以主要以 PHAsset 做为例子）



## 2.从相册选择图片 Asset 或视频 Asset

UIImagePickerController 是从相册选取图片 Asset 和视频 Asset 的选择器，利用其进行图片和视频选择结束之后会通过其代理（实现了 UIImagePickerControllerDelegate 协议）执行下面的方法, 将选择结果返回给用户。

```
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info;
```
从上面的接口看到，选择回来的仅仅是 info 信息，PHAsset 需要利用 info 字典的信息进一步获得。info字典例子：

```
//选择的是图片
info{
    UIImagePickerControllerMediaType = "public.image";
    UIImagePickerControllerOriginalImage = "<UIImage: 0x126cacc60> size {2048, 1365} orientation 0 scale 1.000000";
    UIImagePickerControllerReferenceURL = "assets-library://asset/asset.PNG?i/../B&ext=PNG";
}

//选择的是视频
info{
    UIImagePickerControllerMediaType = "public.movie";
    UIImagePickerControllerMediaURL = "file:///private/../BD-E6D273D5B376.MOV";
    UIImagePickerControllerReferenceURL = "assets-library://asset/asset.MOV?id=546/../B&ext=MOV";
}

//选择的是LivePhoto
info{
    UIImagePickerControllerLivePhoto = "<PHLivePhoto: 0x126e3a170>";
    UIImagePickerControllerMediaType = "com.apple.live-photo";
    UIImagePickerControllerOriginalImage = "<UIImage: 0x126c56b10> size {960, 1280} orientation 0 scale 1.000000";
    UIImagePickerControllerReferenceURL = "assets-library://asset/asset.JPG?id/../B3&ext=JPG";
}
```

从 info 字典的例子可以看到，选择图片，视频和 LivePhoto 三种的回调信息是有区别的，每个结果包含的字段也不相同，但是都有个 __UIImagePickerControllerReferenceURL__ 键值，顾名思义，__assets-library:__ 这条 URL 便是指向我们所选择的 PHAsset 对象的 URL。（LivePhoto 也是属于一种较为特殊的图片 Asset）   

__Fetching Assets__: 从 assets-library URL 获取我们需要的图片和视频 Asset

```
    NSURL *url = [info objectForKey:@"UIImagePickerControllerReferenceURL"];
    PHFetchResult *fetchResult = [PHAsset fetchAssetsWithALAssetURLs:@[url] options:nil];
    PHAsset *asset = fetchResult.firstObject;
```
__Reading Asset Metadata__: PHAsset 对象仅仅包含文件的基本数据 [(Assets contain only metadata)](https://developer.apple.com/library/ios/documentation/Photos/Reference/PHAsset_Class/index.html)。   

这些基本信息包含：媒体属性 (mediaType)，资源类型 (sourceType)，图片像素长宽 (pixelWidth)，拍摄地点（location），视频播放时长 (duration) 等。我们下面的例子用到 __mediaType__ 和 __mediaSubtypes__  两个属性来区分图片，视频和 LivePhoto 三种不同的 Asset。




## 3.将图片 Asset 或视频 Asset 转换为真正的文件

经过上面 Fetching Assets 步骤我们已经成功的从 assets-library url 提取出 PHAsset 对象。现在需要把 PHAsset 转换为真正的视频和图片文件。我们要获取的真正文件无非两种：图片文件和视频文件。上面示例涉及的三种 PHAsset，其中视频 Asset 和图片 Asset 可以分别提取视频和图片文件。LivePhoto Asset 既可以提取图片也可以提取视频。

从 PHAsset 获取图片：

```
+ (void)getImageFromPHAsset:(PHAsset *)asset Complete:(Result)result {
    __block NSData *data;
    PHAssetResource *resource = [[PHAssetResource assetResourcesForAsset:asset] firstObject];
    if (asset.mediaType == PHAssetMediaTypeImage) {
        PHImageRequestOptions *options = [[PHImageRequestOptions alloc] init];
        options.version = PHImageRequestOptionsVersionCurrent;
        options.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
        options.synchronous = YES;
        [[PHImageManager defaultManager] requestImageDataForAsset:asset
                                                          options:options
                                                    resultHandler:
         ^(NSData *imageData,
           NSString *dataUTI,
           UIImageOrientation orientation,
           NSDictionary *info) {
             data = [NSData dataWithData:imageData];
         }];
    }
    
    if (result) {
        if (data.length <= 0) {
            result(nil, nil);
        } else {
            result(data, resource.originalFilename);
        }
    }
}
```

在上面的代码中我们通过判断```asset.mediaType == PHAssetMediaTypeImage ```来区分 PHAsset 是否是一个图片类型的 Asset。值得注意的是 LivePhoto Asset 的 mediaType 属性值也等于 PHAssetMediaTypeImage，所以提取 LivePhoto 里面的图片也可以直接调用此方法。

既然 mediaType 属性一样，怎么才能具体区分一个 PHAsset 是图片 Asset 还是 LivePhoto 呢，答案是通过 PHAsset 的 mediaSubtypes 属性。

PHAsset 的属性和二级属性：

```
typedef NS_ENUM(NSInteger, PHAssetMediaType) {
    PHAssetMediaTypeUnknown = 0,
    PHAssetMediaTypeImage   = 1,
    PHAssetMediaTypeVideo   = 2,
    PHAssetMediaTypeAudio   = 3,
} NS_ENUM_AVAILABLE_IOS(8_0);

typedef NS_OPTIONS(NSUInteger, PHAssetMediaSubtype) {
    PHAssetMediaSubtypeNone               = 0,
    
    // Photo subtypes
    PHAssetMediaSubtypePhotoPanorama      = (1UL << 0),
    PHAssetMediaSubtypePhotoHDR           = (1UL << 1),
    PHAssetMediaSubtypePhotoScreenshot NS_AVAILABLE_IOS(9_0) = (1UL << 2),
    PHAssetMediaSubtypePhotoLive NS_AVAILABLE_IOS(9_1) = (1UL << 3),
    
    // Video subtypes
    PHAssetMediaSubtypeVideoStreamed      = (1UL << 16),
    PHAssetMediaSubtypeVideoHighFrameRate = (1UL << 17),
    PHAssetMediaSubtypeVideoTimelapse     = (1UL << 18),
} NS_AVAILABLE_IOS(8_0);
```

可以看到 PHAsset 一级属性可以区分图片，视频和音频。PhotoLive 属于 Photo 类型下面的一个 subtypes。

从 PHAsset 获取视频：

```
+ (void)getVideoFromPHAsset:(PHAsset *)asset Complete:(Result)result {
    NSArray *assetResources = [PHAssetResource assetResourcesForAsset:asset];
    PHAssetResource *resource;

    for (PHAssetResource *assetRes in assetResources) {
        if (assetRes.type == PHAssetResourceTypePairedVideo ||
            assetRes.type == PHAssetResourceTypeVideo) {
            resource = assetRes;
        }
    }
    NSString *fileName = @"tempAssetVideo.mov";
    if (resource.originalFilename) {
        fileName = resource.originalFilename;
    }
    
    if (asset.mediaType == PHAssetMediaTypeVideo || asset.mediaSubtypes == PHAssetMediaSubtypePhotoLive) {
        PHVideoRequestOptions *options = [[PHVideoRequestOptions alloc] init];
        options.version = PHImageRequestOptionsVersionCurrent;
        options.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
        
        NSString *PATH_MOVIE_FILE = [NSTemporaryDirectory() stringByAppendingPathComponent:fileName];
        [[NSFileManager defaultManager] removeItemAtPath:PATH_MOVIE_FILE error:nil];
        [[PHAssetResourceManager defaultManager] writeDataForAssetResource:resource
                                                                    toFile:[NSURL fileURLWithPath:PATH_MOVIE_FILE]
                                                                   options:nil
                                                         completionHandler:^(NSError * _Nullable error) {
                                                             if (error) {
                                                                 result(nil, nil);
                                                             } else {
                                                                 
                                                                 NSData *data = [NSData dataWithContentsOfURL:[NSURL fileURLWithPath:PATH_MOVIE_FILE]];
                                                                 result(data, fileName);
                                                             }
                                                             [[NSFileManager defaultManager] removeItemAtPath:PATH_MOVIE_FILE  error:nil];
                                                         }];
    } else {
        result(nil, nil);
    }
}
```

注：上面方法兼顾了从 LivePhoto 里面提取视频文件。

## 4.图片或视频文件上传 CDN

上面两段代码具体介绍了 PHAsset 到真正图片文件和视频文件的提取过程。既：可以简单里复用这两个方法来提取真正的 fileData。然后将 fileData 上传到 CDN 或者服务器。

```
typedef void(^Result)(NSData *fileData, NSString *fileName);
+ (void)getImageFromPHAsset:(PHAsset *)asset Complete:(Result)result;
+ (void)getVideoFromPHAsset:(PHAsset *)asset Complete:(Result)result;
```

值得注意的是：上述两个接口，最后回调结果是 fileData。对于图片 PHAsset，因为图片文件不会很大，所以直接拿到图片 data 是可以的。但是对于视频 PHAsset，视频文件较大会占用大量内存空间。 我们可以通过修改上面的接口，用视频的 filePath 来替代 fileData，以解决处理大文件视频情况下的内存占用问题。


修改接口，获取  videoFilePath，注意：使用完成，最好手动删除这个临时文件

```
typedef void(^ResultPath)(NSString *filePath, NSString *fileName);

+ (void)getVideoPathFromPHAsset:(PHAsset *)asset Complete:(ResultPath)result {
    NSArray *assetResources = [PHAssetResource assetResourcesForAsset:asset];
    PHAssetResource *resource;
    
    for (PHAssetResource *assetRes in assetResources) { 
        if (assetRes.type == PHAssetResourceTypePairedVideo ||
            assetRes.type == PHAssetResourceTypeVideo) {
            resource = assetRes;
        }
    }
    NSString *fileName = @"tempAssetVideo.mov";
    if (resource.originalFilename) {
        fileName = resource.originalFilename;
    }
    
    if (asset.mediaType == PHAssetMediaTypeVideo || asset.mediaSubtypes == PHAssetMediaSubtypePhotoLive) {
        PHVideoRequestOptions *options = [[PHVideoRequestOptions alloc] init];
        options.version = PHImageRequestOptionsVersionCurrent;
        options.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
        
        NSString *PATH_MOVIE_FILE = [NSTemporaryDirectory() stringByAppendingPathComponent:fileName];
        [[NSFileManager defaultManager] removeItemAtPath:PATH_MOVIE_FILE error:nil];
        [[PHAssetResourceManager defaultManager] writeDataForAssetResource:resource
                                                                    toFile:[NSURL fileURLWithPath:PATH_MOVIE_FILE]
                                                                   options:nil
                                                         completionHandler:^(NSError * _Nullable error) {
                                                             if (error) {
                                                                 result(nil, nil);
                                                             } else {
                                                                 result(PATH_MOVIE_FILE, fileName);
                                                             }
                                                         }];
    } else {
        result(nil, nil);
    }
}
```

利用返回的 filePath 可以通过流式的读取文件方式，来组织和发送上传请求的 body 体，达到较好的内存占用。同时又拍云 CDN 提供文件分块上传接口，更适合这种大文件的上传操作。



## 5.下载图片和视频保存到手机相册

将图片文件和视频文件保存到手机相册需要以下两个方法：

```
void UIImageWriteToSavedPhotosAlbum(UIImage *image, id completionTarget, SEL completionSelector, void * contextInfo);
void UISaveVideoAtPathToSavedPhotosAlbum(NSString *videoPath, id completionTarget, SEL completionSelector, void * contextInfo);
```

那么如何保存 LivePhoto，对于支持 LivePhoto 的手机用户可能需要将 LivePhoto 保存到手机相册。但是事实上 LivePhoto 不能作为一个整体文件存在于内存硬盘或者服务器。但是可以将一个视频文件和图片文件一起作为 LivePhoto 保存到相册：

保存 LivePhoto 代码示例：

```
    NSURL *photoURL = [NSURL fileURLWithPath:photoURLstring];//@"...picture.jpg"
    NSURL *videoURL = [NSURL fileURLWithPath:videoURLstring];//@"...video.mov"
    
    [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{
        PHAssetCreationRequest *request = [PHAssetCreationRequest creationRequestForAsset];
        [request addResourceWithType:PHAssetResourceTypePhoto
                             fileURL:photoURL
                             options:nil];
        [request addResourceWithType:PHAssetResourceTypePairedVideo
                             fileURL:videoURL
                             options:nil];
        
    } completionHandler:^(BOOL success,
                          NSError * _Nullable error) {
        if (success) {
            [self alertMessage:@"LivePhotos 已经保存至相册!"];

        } else {
            NSLog(@"error: %@",error);
        }
    }];
```


## 6.最后

ALAsset/PHAsset 是属于 iPhone 相册相关操作范围内的概念，ALAsset/PHAsset 并不是文件，不能直接上传 CDN。上传 CDN 需要的真正图片视频文件可以用上文提供的方法从 PHAsset 提取出来。
LivePhoto 属于一种特殊的 PHAsset，可以从 LivePhoto 里面分别提取图片和视频文件之后，再上传CDN。