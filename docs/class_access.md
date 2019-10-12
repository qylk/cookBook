反编译class出现access$xxx类函数的说明

```java
public class TestOuter {
    private int a;
    private int b;

    public TestOuter() {
    }

    private void fun() {
        ++this.a;
    }

    class TestInner {
        int x = 0;

        TestInner() {
            TestOuter.this.b = 1;
            TestOuter.this.a = 0;
            TestOuter.this.fun();
        }
    }
}
```
TestInner 是 TestOuter 的内部类，生成的TestInner.class如下：
```java
class TestOuter$TestInner {
    int x;

    TestOuter$TestInner(TestOuter this$0) {
        this.this$0 = this$0;
        this.x = 0;
        TestOuter.access$002(this$0, 1);
        TestOuter.access$102(this$0, 0);
        TestOuter.access$200(this$0);
    }
}

```

可以看出：
1. TestInner会隐式引用外层的TestOuter.this对象，放在this$0字段中。
2. 为了使内部类访问外部类的私有成员，编译器生成了形似 "外部类.access$XYZ"的函数，XYZ为数字。

        X是按照私有成员在内部类出现的顺序递增的
        YZ为02的话，标明是基本变量成员
        YZ为00的话标明是对象成员或者函数


