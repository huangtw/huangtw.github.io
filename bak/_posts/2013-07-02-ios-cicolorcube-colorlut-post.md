---
layout: post
title: "[iOS] 使用CIColorCube快速製作濾鏡"
description: ""
category: 
tags: []
---
{% include JB/setup %}

如果想要再iOS的App內製作照片濾鏡，可以使用Core Image Framework內一系列的CIFilter來實作。CIFilter提供了可以調整亮度、對比等等各種特效的濾鏡。不過如果能直接用Photoshop等影像處理軟體，直接調整這些色彩參數，可以大幅加快開發速度。

本文將介紹透過CIFilter中的CIColorCube來套用Color Lookup Table，快速開發色彩濾鏡。
<!--more-->
本文的程式原始碼可在以下網址取得   
<https://github.com/huangtw/ColorLUT>

###Color Lookup Table
在影像處理的領域中，當我們想要調整一個影像的色彩時，時常會用到Color Lookup Table(簡稱ColorLUT)的技術。

舉個簡單的例子，如果我們想要讓影像中每個像素的R值變成原本的0.3倍，最基本的作法就是把每一個像素的R值乘以0.3，假設影像的大小為1024x768，那麼總共要786432次浮點數乘法。

如果我們一開始先建一張表，把所有色彩值經過處理(R值變為0.3倍)之後的結果記錄起來，然後把每個像素的色彩值拿去查表，得到處理之後的色彩值，那麼我們只要做786432查表動作，會比浮點運算快上許多。實際上大部分色彩調整的演算法都比這個例子複雜許多，因此更能凸顯出查表法的高效率。

不過如果要把所有色彩的處理結果都存起來，那這張表勢必會相當肥大。以RGB 24 bits為例，每個像素佔3 byte，而總共會有16777216(256x256x256)種色彩，總共佔48MB(256x256x256x3 bytes)，看來並不小。因此實務上我們並不會記下所有的色彩，而是只記下部分的色彩，其他不在表內的色彩則用內插法取得處理後的結果。

因為每個像素的色彩都是由RGB三種顏色組成，因此我們會以三維陣列的方式來儲存這張表。如果把三維陣列中的每一個像素想像成三度空間中的一個點，而R、G、B分別代表X、Y、Z的座標，則陣列中的所有像素可以構成一個正立方體。以RGB 24bits為例，由於R、G、B的值為0~255，因此正方體的長寬高為255x255x255。當我們要查某個像素經過處理之後的色彩，只要將該像素處理前的RGB值當做X、Y、Z座標，位於那個位置上的像素則為處理後的顏色。   
<img src="/images/ios_cicolorcube_colorlut/cube.jpg"/> 

如前面提到的，為了節省大小，我們不可能提供所有的點，因此我們會使用內差法來推算。以4x4x4的ColorLUT為例，X、Y、Z三個方向都只有在0、85、170、255這些位置有資料，如果座標不在這些點上，則必須跟週圍的點做內差來得到色彩的近似值，由於是在三維空間內，因此可以透過三線性內差(Trilinear Interpolation)來計算。

###CIColorCube
在iOS App內可以透過Core Image Framework內的CIColorCube實作Color Lookup Table。使用CIColorCube時必須設定：

- **inputCubeDimension:** 也就是正立方體每邊包含幾個點。以4x4x4的ColorLUT為例，則inputCubeDimension=4
- **inputCubeData:** NSData物件，儲存每個點的RGBA資訊。每個點的R、G、B、A值皆為float(0~1.0)，點的儲存順序為：從Z=0的平面開始，從Y=0的列開始，儲存X=0~X=255的點，後面接著Y=1的列
，接著再儲存Z=1的平面，以此類推。以4x4x4的ColorLUT為例，順序為(0,0,0)、(85,0,0)、(170,0,0)、(255,0,0)、(0,85,0)、(85,85,0)...(0,255,255)、(85,255,255)、(170,255,255)、(255,255,255)

那我們怎麼產生inputCubeData內的資料呢？我們可以先將每個點原本對應的顏色先存在一張圖片內，然後透過Photohop之類的影像處理軟體，套用各種色彩特效到這張圖片，之後App讀取調整過後的圖片，根據每個像素的位置，將色彩資訊填進inputCubeData，之後套用CIColorCube的影像就會跟直接透過Photoshop調整出來的色彩有一樣的效果。

這樣講實在有點抽象，我們以4x4x4的ColorLUT為例，經過Photoshop調整色彩前如下：   
<img src="/images/ios_cicolorcube_colorlut/colorLUT_4x4x4_before.jpg"/>   
可以看到這是一張8x8的圖片，分別包含了4個 4x4的正方形，以右上角正方形為例，這代表Z=0的平面，而X軸由左至右，Y軸為由上至下，左上角第一個像素代表位於(0,0,0)的點，第二個像素代表位於(85,0,0)的點，以此類推，由於這些像素代表的是未處理前的顏色，因此第一個像素的RGB值為0,0,0，第二的像素的RGB值為85,0,0

接著我們將這張圖經過Photoshop降低色彩飽和度，結果如下：   
<img src="/images/ios_cicolorcube_colorlut/colorLUT_4x4x4_after.jpg"/>   
這張圖內的每個像素，代表每個座標點處理之後的色彩。也就是說，如果我們想知道一個RGB值為85,0,0的像素降低飽和度之後的顏色，可從正立方體中位於(85,0,0)的點得知，也就是這張圖中左上角第二個像素的顏色，RGB值為68,16,16。

因此我們只要將第二張圖片中每個像素的RGB值換算成浮點數之後，依序填入NSData中，產生的CIColorCube就可以做出跟Photoshop中一樣的色彩濾鏡。

###CIFilter+ColorLUT
由於使用CIColorCube製作ColorLUT過程有點繁瑣，因此我透過Objective-C的Category機制替CIFiliter擴充一個class method來快速產生ColorLUT

- **CIFilter+ColorLUT.h**
{% highlight objc %}
#import <CoreImage/CoreImage.h>

@interface CIFilter (ColorLUT)

+ (CIFilter *)colorCubeWithColrLUTImageNamed:(NSString *)imageName dimension:(NSInteger)n;

@end
{% endhighlight %}

透過**+ (CIFilter \*)colorCubeWithColrLUTImageNamed:(NSString \*)imageName dimension:(NSInteger)n**這個class method，我們只要提供colorLUT的檔名imageName，以及立方體的邊長n，它就會自動讀取圖檔，將圖檔內的每一個像素填入NSData中，回傳一個由此ColorLUT產生的CIColorCube

- **CIFilter+ColorLUT.m**
{% highlight objc %}
@implementation CIFilter (ColorLUT)

+ (CIFilter *)colorCubeWithColrLUTImageNamed:(NSString *)imageName dimension:(NSInteger)n
{
    UIImage *image = [UIImage imageNamed:imageName];

    int width = CGImageGetWidth(image.CGImage);
    int height = CGImageGetHeight(image.CGImage);
    int rowNum = height / n;
    int columnNum = width / n;

    //檢查圖片大小，必須是n個n*n的正方形，所以長跟寬都必須是n的倍數
    if ((width % n != 0) || (height % n != 0) || (rowNum * columnNum != n))
    {
        NSLog(@"Invalid colorLUT");
        return nil;
    }

    //要讀取每個pixel的RGBA值，必須先把UIImage轉成bitmap  
    unsigned char *bitmap = [self createRGBABitmapFromImage:image.CGImage];
    
    if (bitmap == NULL)
    {
        return nil;
    }

    //將每個pixel的RGBA值填入data中
    //圖片由n個正方形構成，每個正方形代表不同Z值的XY平面
    //正方形內每個pixel的X值有左至右遞增，Y值由上至下遞增
    //而每個正方形所代表的Z值由左至右、由上而下遞增
    int size = n * n * n * sizeof(float) * 4;
    float *data = malloc(size);
    int bitmapOffest = 0;
    int z = 0;
    for (int row = 0; row <  rowNum; row++)
    {
        for (int y = 0; y < n; y++)
        {
            int tmp = z;
            for (int col = 0; col < columnNum; col++)
            {
                for (int x = 0; x < n; x++) {
                    float r = (unsigned int)bitmap[bitmapOffest];
                    float g = (unsigned int)bitmap[bitmapOffest + 1];
                    float b = (unsigned int)bitmap[bitmapOffest + 2];
                    float a = (unsigned int)bitmap[bitmapOffest + 3];
                    
                    int dataOffset = (z*n*n + y*n + x) * 4;

                    data[dataOffset] = r / 255.0;
                    data[dataOffset + 1] = g / 255.0;
                    data[dataOffset + 2] = b / 255.0;
                    data[dataOffset + 3] = a / 255.0;

                    bitmapOffest += 4;
                }
                z++;
            }
            z = tmp;
        }
        z += columnNum;
    }

    free(bitmap);

    //產生一個CIColorCube，並且將data包裝成NSData設進CIColorCube中
    CIFilter *filter = [CIFilter filterWithName:@"CIColorCube"];
    [filter setValue:[NSData dataWithBytesNoCopy:data length:size freeWhenDone:YES] forKey:@"inputCubeData"];
    [filter setValue:[NSNumber numberWithInteger:n] forKey:@"inputCubeDimension"];

    return filter;
}

+ (unsigned char *)createRGBABitmapFromImage:(CGImageRef)image
{
    CGContextRef context = NULL;
    CGColorSpaceRef colorSpace;
    unsigned char *bitmap;
    int bitmapSize;
    int bytesPerRow;
    
    size_t width = CGImageGetWidth(image);
    size_t height = CGImageGetHeight(image);
    
    bytesPerRow   = (width * 4);
    bitmapSize     = (bytesPerRow * height);
    
    bitmap = malloc( bitmapSize );
    if (bitmap == NULL)
    {
        return NULL;
    }
    
    colorSpace = CGColorSpaceCreateDeviceRGB();
    if (colorSpace == NULL)
    {
        free(bitmap);
        return NULL;
    }
    
    context = CGBitmapContextCreate (bitmap,
                                     width,
                                     height,
                                     8,
                                     bytesPerRow,
                                     colorSpace,
                                     kCGImageAlphaPremultipliedLast);
    
    CGColorSpaceRelease( colorSpace );
    
    if (context == NULL)
    {
        free (bitmap);
    }
    
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), image);  
    CGContextRelease(context);
    
    return bitmap;
}

@end
{% endhighlight %}

- **ColorLUTViewController.m**
{% highlight objc %}
#import "ColorLUTViewController.h"
#import "CIFilter+ColorLUT.h"

@interface ColorLUTViewController ()
@property (nonatomic, weak) IBOutlet UIImageView *imageView1;
@property (nonatomic, weak) IBOutlet UIImageView *imageView2;
@end

@implementation ColorLUTViewController

- (void)viewDidLoad
{
    [super viewDidLoad];

    UIImage *image = [UIImage imageNamed:@"sunflower.jpg"];

    //使用colorLUTProcessed.png產生一個CIColorCube
    CIFilter *colorCube = [CIFilter colorCubeWithColrLUTImageNamed:@"colorLUTProcessed.png" dimension:64];

    //設定要處理的圖片
    CIImage *inputImage = [[CIImage alloc] initWithImage: image];
    [colorCube setValue:inputImage forKey:@"inputImage"];

    //取得處理後的圖片，不過此時還沒真的處理，先取得outputImage稍後使用
    CIImage *outputImage = [colorCube outputImage];

    CIContext *context = [CIContext contextWithOptions:[NSDictionary dictionaryWithObject:(__bridge id)(CGColorSpaceCreateDeviceRGB()) forKey:kCIContextWorkingColorSpace]];

    //此時才真的產生處理之後的圖片
    UIImage *newImage = [UIImage imageWithCGImage:[context createCGImage:outputImage fromRect:outputImage.extent]];

    [self.imageView1 setImage:image];
    [self.imageView2 setImage:newImage];
}

@end
{% endhighlight %}

<img src="/images/ios_cicolorcube_colorlut/sunflower.jpg"/>    
左邊為Photoshop調整之前的圖片，右邊為Photoshopt調整顏色之後的照片

接著我們將一樣的色彩濾鏡套用到64x64x64的colorLUT圖

<img src="/images/ios_cicolorcube_colorlut/colorLUT_64x64x64.jpg"/>
左邊為套用Photoshop色彩濾鏡之前的colorLUT，右邊為套用色彩濾鏡之後的colorLUT

<img src="/images/ios_cicolorcube_colorlut/screenshot.jpg" width = "50%" height = "50%"/>   
這是透過CIColorCube處理的結果。可以看出效果跟Photoshop處理的幾乎一模一樣

透過這種方式來開發濾鏡，可以使用美術人員熟悉的Photoshop等軟體調整濾鏡的色彩，在直接套到colorLUT圖之後即可供程式使用，非常方便

###References
- [Core Image Filter Reference](http://developer.apple.com/library/mac/#documentation/GraphicsImaging/Reference/CoreImageFilterReference/Reference/reference.html)
- [Subclassing CIFilter: Recipes for Custom Effects](http://developer.apple.com/library/mac/#documentation/GraphicsImaging/Conceptual/CoreImaging/ci_filer_recipes/ci_filter_recipes.html)
- [Using Lookup Tables to Accelerate Color Transformations](http://http.developer.nvidia.com/GPUGems2/gpugems2_chapter24.html)
