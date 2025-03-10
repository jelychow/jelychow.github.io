---
theme: smartblue
excerpt: "interface vs interface 性能大比拼"
---
# fun interface vs interface 谁的性能更好点？
### 背景介绍
前段时间写了一篇文章 [眼花缭乱的 Kotlin 类型](https://juejin.cn/post/7458931012854595610)，介绍了 fun interface 这个最近在 Now in Android 项目里面出镜率比较高的 kotlin 类型。这时候突然想到了那个常见的问题，哪个性能更好点？

```kt
fun interface Printer {
    fun print(text: String)
}

val huaweiPrinter = object :Printer{
    override fun print(text: String) {
        println("Xiaomi Printer is printing $text")
    }
}

val xiaomiPrinter = Printer{
    println("Xiaomi Printer is printing $it")
}

fun usePrinter(text: String,printer: Printer) {
    printer.print(text)
}

fun main() {
    usePrinter("good",xiaomiPrinter)
    usePrinter("not bad"){
        print("this printer is $it")
    }
    usePrinter("yao yao ling xian",huaweiPrinter::print)
    usePrinter("yao yao ling xian",Printer(huaweiPrinter::print))
    usePrinter("excellent",Printer{
        print("normal printer is printing $it")
    })
}
```

### 反编译
```java
private static final Printer huaweiPrinter = new Printer() {
   public void print(String text) {
      String var2 = "Xiaomi Printer is printing " + text;
      System.out.println(var2);
   }
};

private static final Printer xiaomiPrinter = SealedTestKt::xiaomiPrinter$lambda$0;

public static final Printer getHuaweiPrinter() {
   return huaweiPrinter;
}

public static final Printer getXiaomiPrinter() {
   return xiaomiPrinter;
}

public static final void usePrinter(@NotNull String text, @NotNull Printer printer) {
   printer.print(text);
}

public static final void main() {
   usePrinter("good", xiaomiPrinter);
   usePrinter("not bad", SealedTestKt::main$lambda$1);
   Printer var0 = huaweiPrinter;
   usePrinter("yao yao ling xian", new Printer(var0) {
      final Printer $tmp0;

      {
         this.$tmp0 = $tmp0;
      }

      public final void print(String p0) {
         this.$tmp0.print(p0);
      }
   });
   var0 = huaweiPrinter;
   usePrinter("yao yao ling xian", new Printer(var0) {
      final Printer $tmp0;

      {
         this.$tmp0 = $tmp0;
      }

      public final void print(String p0) {
         this.$tmp0.print(p0);
      }
   });
   usePrinter("excellent", SealedTestKt::main$lambda$2);
}

private static final void xiaomiPrinter$lambda$0(String it) {
   String var1 = "Xiaomi Printer is printing " + it;
   System.out.println(var1);
}

private static final void main$lambda$1(String it) {
   String var1 = "this printer is " + it;
   System.out.print(var1);
}

private static final void main$lambda$2(String it) {
   String var1 = "normal printer is printing " + it;
   System.out.print(var1);
}
```

可以看到上面如果使用经典的匿名内部类的方式，编译器是会生成一个对象，然后在调用的地方也会生成一个对象，对象里面其实我删除部分 equals 和 hashcode 等方法，可见整体上他的调用成本还是挺高的。
使用 lambda 的方式，编译器会将代码编译成一个静态方法，然后调用的时候直接调用这个方法，成本很低。

综合对比下来 fun interface 有如下几个优势
#### 1. **SAM（Single Abstract Method）转换**

-   **SAM转换**：Kotlin 支持 SAM 转换，这意味着可以将 `fun interface` 实例化为 Lambda 表达式。编译器会自动将 Lambda 表达式转换为 `fun interface` 的实例。这种转换比创建一个完整的类实例要轻量得多。

#### 2. **内存开销**

-   **更少的对象创建**：使用 `fun interface` 时，创建的对象通常是轻量级的，因为它们可以直接通过 Lambda 表达式实现，而不需要生成额外的类文件。这减少了内存开销。

#### 3. **性能优化**

-   **直接调用**：由于 `fun interface` 只有一个抽象方法，编译器可以对其进行优化，直接生成调用代码，减少了方法调用的开销。普通接口可能需要额外的虚拟方法调用。

#### 4. **简化的实现**

-   **减少样板代码**：使用 `fun interface`，你可以直接用 Lambda 表达式来实现接口，而不需要写一个完整的实现类。这不仅提高了代码的可读性，还减少了编译和运行时的开销。
### fun interface 的正确打开方式
在实际使用中优先使用 lambda 的方式来使用 fun interface，这样才能充分利用编译器的优化操作。当然这世界上没有万能药，开发过程还是要根据实际情况灵活调整，才能写出优美的代码。