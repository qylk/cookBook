\<? extends T> 和 \<? super T> 是Java泛型中的“通配符（Wildcards）”和“边界（Bounds）”的概念。

* \<? extends T> 指 “上界通配符”  
* \<? super T>   指 “下界通配符”

泛型通配符主要解决泛型容器不兼容问题，比如
```
error: incompatible types: List<Apple> cannot be converted to List<Fruit>
```
即便Apple是Fruit的子类，List\<Apple>也并非List\<Fruit>的子类型，因此无法类型转换，发生以上异常。

编译器简单认为：

* 苹果 IS-A 水果
* 装苹果的篮子 NOT-IS-A 装水果的篮子


但是List\<Apple>却是List\<? extend Fruit>的子类型，因而可以类型转换，这就是通配符出现的原因。

上界通配符\<? extend Fruit>中的 '?' 代表了Fruit的某个子类，但是具体是哪个子类，实际上是未知的，这表示了这个 '?' 类型有一个上界类型Fruit。

这使得List\<? extend Fruit>中的元素可以被当成Fruit类型来访问，但是另一方面却无法向List\<? extend Fruit>中添加任何新的元素(因为你不知道它具体装的是哪种Fruit，所以不能随便加，不过取是没问题的，取出来一定是?类型的。

下界通配符List\<? super Apple>，List\<Fruit>是它的子类型，'?' 代表了Apple的某个超类，但是具体是哪个超类，实际上也是未知的，这表示了这个 '?' 类型有一个下界类型Apple。

这使得可以向List<? super Apple>中新添加Apple，但是另一方面却无法从这个List\<? super Apple>中读取任何一个元素并当成Apple来访问(因为你不知道它具体是Apple的哪个超类，或许是Fruit，或许是Food，你根本不能把它们当作Apple来处理，这与上面是有区别的)。

最后还有一种**无界通配符**，如List\<?>，List\<Object>是它的子类型，不能向List<?>里添加任何新元素，从List<?>里取出的元素只能被当成Object类型来处理。