## Item 10:覆盖equals方法时请遵守通用约定

当你重载了equals方法，你必须遵守它的普遍契约。这是从Object规范的契约：equals方法实现了等价关系，它有这些属性：

* 自反性：对于任何非空引用的值x，x.equals(x)的结果必定是true。
* 对称性：对于任何非空引用的值x，y，有且仅当y.equals(x)为true时，x.equals(y)必为true。
* 传递性：对于任何非空引用值x,y,z,如果x.equals(y)返回true，且y.equals(z)返回true，则x.euqals(z)必为true。
* 一致性：对于任何非空引用值x，y，如果没有修改equals的比较信息，多次调用x.equals(y)必须一直返回true或一直返回false。
* 任何非空引用值x，x.equals(null)必须返回null。

总而言之：

1. 使用==操作符来检查参数是否是这个对象的引用。如果是，返回true。这只是一个性能优化，但是如果比较可能开销很大，这就是值得的。
2. 使用instanceof操作符来检查参数是否具有正确得类型。如果不是，返回false。通常，正确类型指的是equals方法所在的那个类。有些情况下，是指该类所实现的某个接口。如果一个类实现的一个接口实现了equals契约，以允许跨实现接口的类进行比较，那么使用这个接口作为正确的类型。集合接口比如Set，List，Map和Map.Entry具有该属性。
3. 将参数转换为正确的类型。因为有了先前instanceof的测试，转换才能保证成功。
4. 对于类中每个“重要”字段，检查参数的字段是否与此对象的相应字段匹配。如果所有这些测试都成功，返回true；否则，返回false。

最后的建议：

1. 覆写equals方法时始终覆写hashCode。

2. 不要试图太聪明。如果你只是测试字段是否相等，这并不难保持equals方法。如果你过分追求找到等价关系，很容易就会出错。考虑任何形式的别名通常是个坏主意。例如，File类不应该试图把指向同名的符号链接看作相等。

3. 不要在equals声明中用Object替换其他类型。程序员编写一个看起来像这样的equals方法然后花费数小时的时间明白为什么它不能正常工作的情况并不少见：

```java
// Broken - parameter type must be Object!
public boolean equals(MyClass o) {
	...
}
```

总之，不要覆写equals方法除非你不得不：在很多情况下，从Object继承的实现，完全符合你的要求。如果你确实需要覆写equals，确保比较这个类的所有重要字段，并且要符合那五个约定的情况下进行比较。

在Kotlin中，只要简单的使用data calss即可，编译器会自动获得像equals()，hashCode()等等其他更多的方法。