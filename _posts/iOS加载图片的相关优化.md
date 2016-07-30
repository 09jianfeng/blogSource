---
title: iOS加载图片的相关优化
date: 2016-07-09 10:56:19
tags: [iOS,图片]
---


# 高效地实现带圆角效果的 UICollectionViewCell

## 为 UIImageView 设置裁剪过特定边角的图片

### UIImageView+HGCWebImage.h

```
#import <UIKit/UIKit.h>
#import <SDWebImage/SDWebImageManager.h>
#import <NYXImagesKit/NYXImagesKit.h>

typedef NS_OPTIONS(NSInteger, HGCRectCorner) {
    HGCRectCornerNone           = 0,
    HGCRectCornerTopLeft        = 1 << 0,
    HGCRectCornerTopRight       = 1 << 1,
    HGCRectCornerBottomLeft     = 1 << 2,
    HGCRectCornerBottomRight    = 1 << 3,
    HGCRectCornerAll            = HGCRectCornerTopLeft|HGCRectCornerTopRight|HGCRectCornerBottomLeft|HGCRectCornerBottomRight,
};

@interface UIImageView (HGCWebImage)

- (void)hgc_setImageWithURLString:(NSString *)urlstring placeholderImage:(UIImage *)placeholderImg;
- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg;

/**
 *  sizeToFill 的 width/height 最好都为整数，否则还是会出现像素不对齐的问题
 */

- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg
                  scaleToFillSize:(CGSize)sizeToFill;

- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg
                  scaleToFillSize:(CGSize)sizeToFill
                  roundingCorners:(HGCRectCorner)corners
                      cornerRadii:(CGFloat)radius;

@end
```

### UIImageView+HGCWebImage.m

```
#import "UIImageView+HGCWebImage.h"

@implementation UIImageView (HGCWebImage)

static inline UIImage *HGCWebImage_cropImage(UIImage *srcImage, CGSize size, CGFloat scale, UIRectCorner corners, CGFloat radius)
{
    if (srcImage &&
        CGSizeEqualToSize(size, CGSizeZero) == NO &&
        corners != HGCRectCornerNone &&
        (NSInteger)radius > 0)
    {
        UIGraphicsBeginImageContextWithOptions(size, NO, scale);
        CGRect drawingRect = (CGRect){CGPointZero, size};
        [[UIBezierPath bezierPathWithRoundedRect:drawingRect byRoundingCorners:(UIRectCorner)corners cornerRadii:CGSizeMake(radius, radius)] addClip];
        [srcImage drawInRect:drawingRect];
        UIImage *cropedImage = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        return cropedImage;
    }
    else {
        return nil;
    }
}

- (void)hgc_setImageWithURLString:(NSString *)urlstring placeholderImage:(UIImage *)placeholderImg {
    [self hgc_setImageWithURLString:urlstring placeholderImage:placeholderImg loadFailImage:placeholderImg];
}

- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg
{
    [self hgc_setImageWithURLString:urlstring placeholderImage:placeholderImg loadFailImage:loadFailImg scaleToFillSize:CGSizeZero];
}

- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg
                  scaleToFillSize:(CGSize)sizeToFill
{
    [self hgc_setImageWithURLString:urlstring placeholderImage:placeholderImg loadFailImage:loadFailImg scaleToFillSize:sizeToFill roundingCorners:HGCRectCornerNone cornerRadii:0];
}

- (void)hgc_setImageWithURLString:(NSString *)urlstring
                 placeholderImage:(UIImage *)placeholderImg
                    loadFailImage:(UIImage *)loadFailImg
                  scaleToFillSize:(CGSize)sizeToFill
                  roundingCorners:(HGCRectCorner)corners
                      cornerRadii:(CGFloat)radius
{
    if (loadFailImg == nil) {
        loadFailImg = placeholderImg;
    }
    
    
    /**
     *  首先过滤空白URL
     */
    if (urlstring == nil || [urlstring hgc_isEmpty]) {
        DDLogWarn(@"Image loading : url is empty");
        self.image = loadFailImg;
        
        return;
    }
    
    
    /**
     *  生成读取缓存的key
     *
     *  key的生成规则：
     *  if (size == CGSizeZero)
     *      key = urlstring
     *  else
     *      key = [urlstring]_[size.width]-[size.height]_[corners]
     */
    NSString *keyForCachedImg;
    if (CGSizeEqualToSize(sizeToFill, CGSizeZero)) {
        keyForCachedImg = urlstring;
    }
    else {
        keyForCachedImg = [urlstring stringByAppendingString:[NSString stringWithFormat:@"_%zd-%zd_%zd", (NSInteger)(sizeToFill.width), (NSInteger)(sizeToFill.height), (NSInteger)corners]];
    }
    
    
    /**
     *  读缓存
     */
    UIImage *cachedImage = [[[SDWebImageManager sharedManager] imageCache] imageFromDiskCacheForKey:keyForCachedImg];
    if (cachedImage) {
        self.image = cachedImage;
        
        // 如果缓存存在，就直接返回
        return;
    }
    self.image = placeholderImg;
    
    
    /**
     *  从网络下载最新的图片，然后缓存起来并更新到界面
     */
    [[[SDWebImageManager sharedManager] imageDownloader] downloadImageWithURL:[NSURL URLWithString:urlstring] options:SDWebImageDownloaderAllowInvalidSSLCertificates progress:nil completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
        
        if (image && error == nil && finished) {
            hgc_dispatch_background_async(^{
                UIImage *desImage;
                if (CGSizeEqualToSize(sizeToFill, CGSizeZero)) { // 不需要拉伸和裁剪
                    desImage = image;
                }
                else {
                    CGFloat scale = [[UIScreen mainScreen] scale];
                    
                    // 先拉伸图片
                    desImage = [image scaleToFillSize:CGSizeMake(sizeToFill.width * scale, sizeToFill.height * scale)];
                    
                    // 再按需裁剪图片
                    UIImage *cropedImg = HGCWebImage_cropImage(desImage, sizeToFill, scale, (UIRectCorner)corners, radius);
                    if (cropedImg) {
                        desImage = cropedImg;
                    }
                }
                
                // 缓存起来，并更新到界面
                if (desImage) {
                    [[[SDWebImageManager sharedManager] imageCache] storeImage:desImage forKey:keyForCachedImg toDisk:YES];
                    hgc_dispatch_main_async_safe(^{
                        self.image = desImage;
                    });
                }
                else {
                    DDLogError(@"Download Web Image: des image is nil");
                    hgc_dispatch_main_async_safe(^{
                        self.image = loadFailImg;
                    });
                }
            });
        }
        else {
            DDLogError(@"Download Web Image: error = %@", error);
            hgc_dispatch_main_async_safe(^{
                self.image = loadFailImg;
            });
        }
    }];
}

@end
```

## 实现带圆角效果的 CollectionViewCell 基类（不造成离屏渲染）

### HGCRoundCornerCollectionViewCell.h

```
@interface HGCRoundCornerCollectionViewCell : UICollectionViewCell

@property (nonatomic, assign) CGSize cachedCellSize;

- (void)setup;

- (void)updateCachedCellSize:(CGSize)cellSize;

@end
```

### HGCRoundCornerCollectionViewCell.m

```
#import "HGCRoundCornerCollectionViewCell.h"
#import "UIColor+HGCUtil.h"
#import "UIView+HGCFrameSugar.h"


///////////////////////////////////////////////////////////////////////////////////////////


@interface HGCRoundCornerCollectionViewCell ()

@end

@implementation HGCRoundCornerCollectionViewCell


///////////////////////////////////////////////////////////////////////////////////////////


#pragma mark - Init

- (instancetype)init {
    self = [super init];
    if (self) {
        [self setup];
    }
    return self;
}

- (instancetype)initWithFrame:(CGRect)frame {
    self = [super initWithFrame:frame];
    if (self) {
        [self setup];
    }
    return self;
}

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if (self) {
        [self setup];
    }
    return self;
}

- (void)setup {
    self.backgroundColor = HGCColor_GrayBackground;
    self.opaque = YES;
    self.alpha = 1.0f;
    self.clipsToBounds = YES;
    
    self.cachedCellSize = CGSizeZero;
}


///////////////////////////////////////////////////////////////////////////////////////////


#pragma mark - Override Methods

- (void)setHighlighted:(BOOL)highlighted {
    [super setHighlighted:highlighted];
}

- (void)prepareForReuse {
    [super prepareForReuse];
    
    _cachedCellSize = CGSizeZero;
}

- (void)drawRect:(CGRect)rect {
    CGContextRef context = UIGraphicsGetCurrentContext();
    
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:rect cornerRadius:kHGCDefaultViewLayerCornerRadius];
    CGContextAddPath(context, path.CGPath);
    CGContextClosePath(context);
    
    CGContextSaveGState(context);
    CGContextClip(context);
    CGContextSetFillColorWithColor(context, [UIColor whiteColor].CGColor);
    CGContextFillRect(context, rect);
    CGContextRestoreGState(context);
}


///////////////////////////////////////////////////////////////////////////////////////////


#pragma mark - Common Methods

- (void)updateCachedCellSize:(CGSize)cellSize {
    if (HGCIsViewSizeZero(self)) {
        if (HGCIsSizeZero(cellSize) == NO) {
            self.cachedCellSize = cellSize;
        }
    }
    else {
        self.cachedCellSize = self.frame_size;
    }
}

@end
```






