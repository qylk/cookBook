BitmapFactory#Options中对inSampleSize是这样说明的：
```java
/**
 * If set to a value > 1, requests the decoder to subsample the original
* image, returning a smaller image to save memory. The sample size is
* the number of pixels in either dimension that correspond to a single
* pixel in the decoded bitmap. For example, inSampleSize == 4 returns
* an image that is 1/4 the width/height of the original, and 1/16 the
* number of pixels. Any value <= 1 is treated the same as 1. Note: the
* decoder uses a final value based on powers of 2, any other value will
* be rounded down to the nearest power of 2.
 */
public int inSampleSize;
```
看起来inSampleSize最终对图片的缩放比例只能是2的幂次，也就是说只能缩放出1倍、1/2、1/4、1/8这样子，比如inSampleSize=3时，最终缩放比例可能仍然是1/2，而不是1/3。
通过测试发现在不同系统上，这一点不一定：
* 4.4 缩放到2的 k 次方
* 5.0 缩放到2的 k 次方
* 6.0 缩放到2的 k 次方
* 7.0 给多少倍，是多少倍，即支持1/3, 1/5比例缩放等等