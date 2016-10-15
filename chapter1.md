# Java

## Java8的新特性

> ### Lambda表达式

为简化Lambda表达式应用提供的泛型接口类：
| --- | --- |
| 泛型接口 | 说明 |
| 

Function&lt;T, R&gt;：将 T 作为输入，返回 R 作为输出，他还包含了和其他函数组合的默认方法。

Predicate&lt;T&gt; ：将 T 作为输入，返回一个布尔值作为输出，该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（与、或、非）。

Consumer&lt;T&gt; ：将 T 作为输入，不返回任何内容，表示在单个参数上的操作。

和上述接口同时提出的还有如下一组语法糖：

---

| 说明 | 方法引用 | 等价的lambda表达式 |
| --- | --- | --- |
| 静态方法引用 | String.valueOf | \(x\) -&gt; String.valueOf\(x\); |
| 非静态方法引用 | Object::toString | (x) -> x.toString(); |
| 特定对象的方法引用 | x::toString | () -> x.toString(); |
| 构造函数引用 | ArrayList::new | () -> new ArrayList<>(); |

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

