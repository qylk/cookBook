# mipmap和drawable的区别
mipmap只是放app的icon图标，其他图标和资源仍然推荐放drawable（放在mipmap也可以，没什么区别）

在App内，无论你将图片放在drawable还是mipmap目录，系统只会加载对应density中的图片，例如xxhdpi的设备，只会加载drawable-xxhdpi或者mipmap-xxhdpi中的资源。
而在Launcher中，如果使用mipmap，那么Launcher会自动加载更加合适的密度的启动图标。

# drawable资源图
## 查找顺序
当我们使用资源id来去引用一张图片时，Android会使用一些规则来去帮我们匹配最适合的图片。什么叫最适合的图片？比如我的手机屏幕密度是xxhdpi，那么drawable-xxhdpi文件夹下的图片就是最适合的图片。

因此，当我引用android_logo这张图时，如果drawable-xxhdpi文件夹下有这张图就会优先被使用，在这种情况下，图片是不会被缩放的。

但是，如果drawable-xxhdpi文件夹下没有这张图时， 系统就会自动去其它文件夹下找这张图了，**优先会去更高密度**的文件夹下找这张图片，我们当前的场景就是drawable-xxxhdpi文件夹，然后发现这里也没有android_logo这张图，接下来会尝试再找更高密度的文件夹，发现没有更高密度的了，这个时候会去drawable-nodpi文件夹找这张图，发现也没有，那么就会去更低密度的文件夹下面找，顺序依次是drawable-xhdpi -> drawable-hdpi -> drawable-mdpi -> drawable-ldpi。

查找顺序：当前密度 -> 高密度 -> nodpi -> 低密度

## 密度缩放
当缩放时，会根据系统和使用的文件夹的density进行缩放
```
内存图尺寸 = 资源图大小 * （ 设备显示密度 / 资源所在目录代表的密度 ）
```

所以如果 图片资源所在目录代表的密度 **小于** 设备显示密度，将导致内存图尺寸变大。

所以如果只切一张图，**尽量找设计人员要一张高品质图，并放在高密度文件夹下，这样在低密度屏幕上，放大系数小于1，可以节省图片的内存开支**，如果不注意这个规律，搞反了可能导致内存溢出。

dpi|density
---|---
xxhdpi	|3
xhdpi   |2
hdpi	|1.5
xxxhdpi	|4

假设设备是xxhdpi，资源图原始大小300*300。
* 当把资源图放在xxhdpi目录时，图片显示原始大小300x300。
* 当把资源图放在hdpi目录时，因为该图片在低密度下，系统会认它不够大，会自动帮我们放大， 300 * 3 / 1.5 ==> 600x600
* 当把资源图放在xxxhdpi目录时，300 * 3 / 4 ==> 225x225
