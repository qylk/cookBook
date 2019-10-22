怎样计算一张图片的大小，加载bitmap过程（怎样保证不产生内存溢出），二级缓存，LRUCache算法。
计算一张图片的大小
图片占用内存的计算公式：图片高度 图片宽度 一个像素占用的内存大小.所以，计算图片占用内存大小的时候，要考虑图片所在的目录跟设备密度，这两个因素其实影响的是图片的高宽，android会对图片进行拉升跟压缩。

加载bitmap过程（怎样保证不产生内存溢出）
由于Android对图片使用内存有限制，若是加载几兆的大图片便内存溢出。Bitmap会将图片的所有像素（即长x宽）加载到内存中，如果图片分辨率过大，会直接导致内存OOM，只有在BitmapFactory加载图片时使用BitmapFactory.Options对相关参数进行配置来减少加载的像素。

BitmapFactory.Options相关参数详解
(1).Options.inPreferredConfig值来降低内存消耗。
比如：默认值ARGB_8888改为RGB_565,节约一半内存。
(2).设置Options.inSampleSize 缩放比例，对大图片进行压缩 。
(3).设置Options.inPurgeable和inInputShareable：让系统能及时回 收内存。
A：inPurgeable：设置为True时，表示系统内存不足时可以被回 收，设置为False时，表示不能被回收。
B：inInputShareable：设置是否深拷贝，与inPurgeable结合使用，inPurgeable为false时，该参数无意义。
(4).使用decodeStream代替其他方法。
decodeResource,setImageResource,setImageBitmap等方法