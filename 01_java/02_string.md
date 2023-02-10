### 字符串是不可变
不可变对象是在完全创建后其内部状态保持不变的对象。这意味着，一旦对象被赋值给变量，我们既不能更新引用，也不能通过任何方式改变内部状态。
一个string对象在内存(堆)中被创建出来，他就无法被修改。而且，String类的所有方法都没有改变字符串本身的值，都是返回了一个新的对象。

### 为什么String要设计成不可变
***缓存***
字符串是使用最广泛的数据结构。大量的字符串的创建是非常耗费资源的，所以，Java提供了对字符串的缓存功能，可以大大的节省堆空间。
JVM中专门开辟了一部分空间来存储Java字符串，那就是字符串池。
通过字符串池，两个内容相同的字符串变量，可以从池中指向同一个字符串对象，从而节省了关键的内存资源。

![缓存](https://github.com/chenxh/interviews/blob/main/imgs/string_0.png "图片title")


***安全性***
字符串在Java应用程序中广泛用于存储敏感信息，如用户名、密码、连接url、网络连接等。JVM类加载器在加载类的时也广泛地使用它。
因此，保护String类对于提升整个应用程序的安全性至关重要。
当我们在程序中传递一个字符串的时候，如果这个字符串的内容是不可变的，那么我们就可以相信这个字符串中的内容。
但是，如果是可变的，那么这个字符串内容就可能随时都被修改。那么这个字符串内容就完全不可信了。这样整个系统就没有安全性可言了。

***线程安全***
不可变会自动使字符串成为线程安全的，因为当从多个线程访问它们时，它们不会被更改。

***hashcode缓存***
由于字符串对象被广泛地用作数据结构，它们也被广泛地用于哈希实现，如HashMap、HashTable、HashSet等。在对这些散列实现进行操作时，经常调用hashCode()方法。
不可变性保证了字符串的值不会改变。因此，hashCode()方法在String类中被重写，以方便缓存，这样在第一次hashCode()调用期间计算和缓存散列，并从那时起返回相同的值。

***性能***
前面提到了的字符串池、hashcode缓存等，都是提升性能的提现。
因为字符串不可变，所以可以用字符串池缓存，可以大大节省堆内存。而且还可以提前对hashcode进行缓存，更加高效。
由于字符串是应用最广泛的数据结构，提高字符串的性能对提高整个应用程序的总体性能有相当大的影响。

### JDK6 和 JDK7 的substring 
substring(int beginIndex, int endIndex) 方法截取字符串并返回其[beginIndex,endIndex-1]范围内的内容。
**JDK6 中的实现方式**： 创建一个新的 String 对象，和截取前的 String 对象指向同一个字符数组(String 对象中的value)， offset 和 count 值不一样。
**JDK6 中的问题**：如果一个长字符串，substring 截取很小的一段。这可能导致性能问题，因为你需要的只是一小段字符序列，但是你却引用了整个字符串（因为这个非常长的字符数组一直在被引用，所以无法被回收，就可能导致内存泄露）。改进方式：
```
x = x.substring(x, y) + ""
```
**JDK7 中的改进**：新的 String 对象指向一个新的字符数组。 

### replaceFirst、replaceAll、replace

replace(CharSequence target, CharSequence replacement) ，用replacement替换所有的target，两个参数都是字符串。
replaceAll(String regex, String replacement) ，用replacement替换所有的regex匹配项，regex很明显是个正则表达式，replacement是字符串。
replaceFirst(String regex, String replacement) ，基本和replaceAll相同，区别是只替换第一个匹配项。

### String对“+”的重载
Java 中 String 的 “+” 只是一个语法。编译之后的等同于
```
(new StringBuilder()).append(wechat).append(",").append(introduce).toString();
``` 
特殊情况, String s = "a" + "b"
```
String s = "a" + "b";
// 编译后
String s = "ab";
```

### 字符串拼接的方法
1、如果只是简单的字符串拼接，考虑直接使用"+"即可。
2、如果是在for循环中进行字符串拼接，考虑使用StringBuilder（线程安全）和StringBuffer。
3、如果是通过一个List进行字符串拼接，则考虑使用StringJoiner。 StringJoiner其实是通过StringBuilder实现的，所以他的性能和StringBuilder差不多，他也是非线程安全的。

### 字符串池
在JVM中，为了减少相同的字符串的重复创建，为了达到节省内存的目的。会单独开辟一块内存，用于保存字符串常量，这个内存区域被叫做字符串常量池。
在JDK 7以前的版本中，字符串常量池是放在永久代中的。
在 JDK 8中，彻底移除了永久代，使用元空间替代了永久代，于是字符串常量池再次从堆内存移动到永久代中。

### Class 常量池
 Class常量池可以理解为是Class文件中的资源仓库。 Class文件中除了包含类的版本、字段、方法、接口等描述信息外，还有一项信息就是常量池(constant pool table)，用于存放编译器生成的各种字面量(Literal)和符号引用(Symbolic References)。
由于不同的Class文件中包含的常量的个数是不固定的，所以在Class文件的常量池入口处会设置两个字节的常量池容量计数器，记录了常量池中常量的个数。
-w697
常量池中主要存放两大类常量：字面量（literal）和符号引用（symbolic references）。
Class是用来保存常量的一个媒介场所，并且是一个中间场所。在JVM真的运行时，需要把常量池中的常量加载到内存中。

![](https://github.com/chenxh/interviews/blob/main/imgs/string_1.png "")

### 运行时常量池
运行时常量池（ Runtime Constant Pool）是每一个类或接口的常量池（ Constant_Pool）的运行时表示形式。
运行时常量池中包含了若干种不同的常量：
编译期可知的字面量和符号引用（来自Class常量池） 运行期解析后可获得的常量（如String的intern方法）
所以，运行时常量池中的内容包含：Class常量池中的常量、字符串常量池中的内容

### String.inern
当代码中出现双引号形式（字面量）创建字符串对象时，JVM 会先对这个字符串进行检查，如果字符串常量池中存在相同内容的字符串对象的引用，则将这个引用返回；否则，创建新的字符串对象，然后将这个引用放入字符串常量池，并返回该引用。
除了以上方式之外，还有一种可以在运行期将字符串内容放置到字符串常量池的办法，那就是使用intern 。

在每次赋值的时候使用 String 的 intern 方法，如果常量池中有相同值，就会重复使用该对象，返回对象引用。

### String 长度限制
字符串有长度限制，在编译期，要求字符串常量池中的常量不能超过65535，并且在javac执行过程中控制了最大值为65534。
在运行期，长度不能超过Int的范围，否则会抛异常。







