
---
layout: post
title:  "WebP 格式图片在 iOS 平台的应用"
date:   2017-03-15 18:15:00
author: rotoava
comments: true
---





> 移动端应用往往会有大量的图片展示场景，图片的加载速度对用户体验至关重要。WebP 作为一种更高效的图片编码格式可以显著的减小图片体积，提升图片加载速度。随着各存储及 CDN 厂商的支持，在 iOS 平台上使用 WebP 图片也越来越方便。




### 1.  WebP 格式简介

WebP 是 一种新型图片格式，具备无损和有损两种压缩方式，同时还支持透明通道和动图效果。WebP 最显著的特征是其高效的压缩效率，普遍的，将 gif 和 png 图片转换为 WebP 格式可以减小 80% 的文件体积。将 jpg 图片转换为 WebP 格式可减小 50％ 的文件体积。[对比测试页面](https://www.upyun.com/webp.html)





###  2 . WebP 格式的编码和解码

本文主要介绍 WebP 在 iOS 平台的使用，首先我们简单了解一下 WebP 编码和解码过程。 

图片的编码过程是将 YUV 或者 RGB 像素格式的原始图片编码为压缩格式如 WebP 或者 jpg。图片的解码过程是将压缩格式还原为 YUV 或者 RGB 的原始图片。图片在控件或者屏幕上的渲染展示只能使用原始像素格式，也就是压缩格式的图片必须解码之后才能展示。

我们参考 [iOS-WebP](https://github.com/seanooi/iOS-WebP) 工程，简单了解一下 WebP 的编解码流程。


解码过程：

```
//UIImage+WebP.m


+ (UIImage *)imageWithWebPData:(NSData *)imgData error:(NSError **)error {
    // `WebPGetInfo` weill return image width and height
    int width = 0, height = 0;
    if(!WebPGetInfo([imgData bytes], [imgData length], &width, &height)) {
        ...
        return nil;
    }
    
    WebPDecoderConfig * config = malloc(sizeof(WebPDecoderConfig));
    if(!WebPInitDecoderConfig(config)) {
        ...
        return nil;
    }
    
    config->options.no_fancy_upsampling = 1;
    config->options.bypass_filtering = 1;
    config->options.use_threads = 1;
    config->output.colorspace = MODE_RGBA;
    
    // Decode the WebP image data into a RGBA value array
    VP8StatusCode decodeStatus = WebPDecode([imgData bytes], [imgData length], config);
    if (decodeStatus != VP8_STATUS_OK) {
        ...
        return nil;
    }
    
    // Construct UIImage from the decoded RGBA value array
    uint8_t *data = WebPDecodeRGBA([imgData bytes], [imgData length], &width, &height);
    ...
    CGImageRef imageRef = CGImageCreate(width, height, 8, 32, 4 * width, colorSpaceRef, bitmapInfo, provider, NULL, YES, renderingIntent);
    UIImage *result = [UIImage imageWithCGImage:imageRef];
    ...
    
    return result;
}


```

可见这个解码函数的作用是将 WebP 图片 Data 转换成 UIImage。整个解码功能的实现是依赖于 ```libwebp ``` 库。解码第一步是调用 WebPGetInfo 函数从 WebpData 中读取图片长宽等信息。第二步是配置解码参数 WebPDecoderConfig ，其中一个重要的参数是  output.colorspace ，即设置解码之后图片的像素格式。示例代码中设置为  MODE_RGBA，从 enum WEBP_CSP_MODE 类型的定义看，解码设置支持多种的 RGB 和 YUV 类型格式。另外的还可以配置图片剪裁、缩放等参数，具体可看考[libwebp API](https://developers.google.com/speed/webp/docs/api)。 接下来，第三步将 webP 数据解码为 RGBA 数据， 第四步将 RGBA 组装为 UIImage。


编码过程与解码相似，需要注意的是 WebP 的有损压缩方式只支持压缩 YUV 数据，因此如果原始格式是 RGB 可以通过转换为 YUV 之后再进行压缩编码。libwebp 也内置了一个转换函数可供使用。

注意：编码过程示例代码仅用于简单的阐述 WebP 编解码过程，在工程实际中解码 WebP 可以复用 SDWebImage 库中的解码方法。编码 WebP 可以调用的云处理 API 在服务器端完成。



###  3 . 使用 UIImageView 展示 WebP 图片


在 UIImageView 中使用 WebP 格式图片的最佳实践是依赖 SDWebImage 库。


安装 SDWebImage（支持 WebP）：

```
pod'SDWebImage/WebP'

```

使用方式：

```  
#import <SDWebImage/UIImageView+WebCache.h>

- (void)viewDidLoad {
    [super viewDidLoad];
    
    
    //动图类型的 WebP
    NSString *animatedWebpurl = @"https://p.upyun.com/demo/webp/webp/animated-gif-5.webp";
    [self.imageView1 sd_setImageWithURL:[NSURL URLWithString:animatedWebpurl]];
    
    
    //静图类型的 WebP
    NSString *webpurl = @"https://p.upyun.com/demo/webp/webp/jpg-1.webp";
    [self.imageView2 sd_setImageWithURL:[NSURL URLWithString:webpurl]];

}


```

SDWebImage 安装之后内置 libwebp 源码，并在  UIImage+WebP.m 封装实现了 WebP 格式的解码。通过 SDWebImage 我们可以非常简单方便的在 UIImageView 中使用 WebP 格式的图片。





###  4 . 使用 UIWebView 展示 WebP 图片

以 Native 方式开发的 app 中也会大量使用的 UIWebView 来展示一些简单页面，然而 Safari 及 UIWebView 当前并不支持 WebP 格式。若是想在 UIWebView 也把图片显示出来，一个解决思路就是：拦截替换。拦截 WebP 图片然后转换为 jpg 或者 png 再交给 UIWebView 进行渲染和展示。


拦截替换 WebP 有两种具体实现方式。		
				
方式 1：    

通过 UIWebView 代理函数给出的页面加载时机进行 WebP 图片的拦截和替换。这种方式首先要遍历获取 HTML 文档中的所有 WebP 图片链接。获取图片链接过程需要依赖第三方 HTML parser 来解析 HTML 文档，或者通过提前协商好的 HTML 内嵌 js 函数进行通信。在得到所有的 WebP 图片链接之后，将 WebP 下载解码转换为 jpg 或者其他图片 data，然后再进行 base64 编码成字符串内嵌到 ```<img>``` 标签，让 UIWebView 进行加载。


方式 2：

通过 NSURLProtocol 全局拦截和过滤所有 HTTP 请求，将 WebP 数据直接转换为其他格式图片数据，再封装为 response body 吐给原来的 HTTP 请求。代码示例：


```  
//WebP_demo/WebPHttpProxy.m
#pragma mark dataDelegate

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data{
    
    NSData *transData = data;
    if ([dataTask.currentRequest.URL.absoluteString hasSuffix:@"webp"]) {
        
        //借用 SDWebImage 中国 WebP 的解码方法，将 webp data 转化为 JPEG data
        UIImage *imgData = [UIImage sd_imageWithWebPData:data];
        transData = UIImageJPEGRepresentation(imgData, 1.0f);
    }
    
    [self.client URLProtocol:self didLoadData:transData];
}


```

这样就可以达到以透明的方式使用 WebP 图片。但是由于 NSURLProtocol 的全局性质，影响范围大，这种方式存在潜在的风险。需要严格的过滤和限制 WebP 请求的拦截。NSURLProtocol 作用的叠加性质，也无法保证与其它第三方代码的兼容。虽然这种方式有很多代码可以参考，本文附带的 demo 也有此种方式的实现，但是在具体工程中使用还需要谨慎和完善。


###  5 . WebP 的多媒体 API


经过简单的开发，UIWebView 及 UIImageView 都可以进行 WebP 图片的展示，所以对于不需要兼顾 web 端的应用完全可以抛弃 jpg 及 gif 等格式，来全面使用 WebP 图片格式。但对于有 web 端的应用还需要保留通用图片格式以兼容 safari 浏览器。即使在这种情形下 Native app 也可以通过云处理接口来享用 WebP 格式带来的高效。

例子：通过又拍云 url 作图 API 来快捷的使用 WebP 格式图片。

```
//WebP_demo/ViewController3.h

    [self.imageView1 sd_setImageWithURL:[NSURL URLWithString:@"https://test86400.b0.upaiyun.com/ios_sdk_new/6.gif!/format/webp"]];
    
    [self.imageView2 sd_setImageWithURL:[NSURL URLWithString:@"https://test86400.b0.upaiyun.com/ios_sdk_new/picture.jpg!/format/webp"]];

```

通过在原始图片 url 后面拼接后缀 ``` !/format/webp ``` 即可以实时的返回 WebP 图片。 又拍云 url 作图 API 支持将 jpg、png、gif 等多种格式图片实时的转化为 WebP 并且缓存。通过 SDWebImage 结合 url 作图 API 可以简单方便的在 iOS 工程中使用 WebP 图片，几乎不需要任何架构的改变就可以大幅度减小图片请求流量。例如上面的 ``` 6.gif ``` 动图经过 WebP 格式的后缀拼接之后，HTTP 请求的流量减小了 89.8 %。 
 


###  6 . 总结

WebP 格式图片具备较高的压缩效率同时也支持动图效果，其完全可以替代 jpg、png、gif 等格式图片成为网络图片格式的首选。在 iOS 平台使用 SDWebImage 结合云处理 API 可以简单快捷的使用 WebP 格式图片，大幅度减少图片请求流量，提高图片加载体验。 



### 相关链接



* [iOS 客户端基于 WebP 图片格式的流量优化](https://segmentfault.com/a/1190000006266276)
* [在WebView中使用webp格式图片](http://blog.csdn.net/u010124617/article/details/51545611) 
* [在iOS项目中使用WebP格式图片](http://blog.devzeng.com/blog/ios-webp-usage.html)
* [WebP 探寻之路](https://isux.tencent.com/introduction-of-webp.html)
* [WebP API Documentation](https://developers.google.com/speed/webp/docs/api)
* [WebP-iOS-demo](http://test86400.b0.upaiyun.com/WebP-iOS-demo.zip)






