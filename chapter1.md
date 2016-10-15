# Java

## Java8的新特性

> ### Lambda表达式

为简化Lambda表达式应用提供的泛型接口类：

| f泛型接口 | 说明 |
| --- | --- |
| Function&lt;T, R&gt; | 将 T 作为输入，返回 R 作为输出 |
| Predicate&lt;T&gt; | 将 T 作为输入，返回一个布尔值 |
| Consumer&lt;T&gt; | 将 T 作为输入，不返回任何内容 |
| Supplier&lt;T, R&gt; | 没有任何输入，返回T |
| BiConsumer&lt;T, U&gt; | T, U做为输入，不返回任何内容 |

除此上述罗列的用法之外，还包括一组类似的泛型接口声明，详细参见[java.util.function](http://javadocs.techempower.com/jdk18/api/java/util/function/package-summary.html)

和上述接口同时提出的还有如下一组语法糖：

| 说明 | 方法引用 | 等价的lambda表达式 |
| --- | --- | --- |
| 静态方法引用 | String.valueOf | \(x\) -&gt; String.valueOf\(x\); |
| 非静态方法引用 | Object::toString | \(x\) -&gt; x.toString\(\); |
| 特定对象的方法引用 | x::toString | \(\) -&gt; x.toString\(\); |
| 构造函数引用 | ArrayList::new | \(\) -&gt; new ArrayList&lt;&gt;\(\); |
这两组语法组合使用可发挥Lambda的最大威力。

> ### 接口增强

default关键字为接口提供默认实现

```



public interface DefaultFunInterface {

 // 定义默认方法 count

 default int count() {

 return 1;

 }

 }



```

为接口增加静态方法，该静态方法可直接通过接口调用

\`\`\`

public interface StaticFunInterface {

public static int find\(\) {

return 1;

}

}

\`\`\`

> ### 集合的Stream操作

串行流与并行流

stream\(\)默认为串行流

parallelStream\(\)默认为并行流

stream\(\).sequential\(\) 返回串行的流

stream\(\).parallel\(\) 返回并行的流

中间操作：

