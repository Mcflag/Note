## Item 1:考虑以静态工厂方法代替构造函数

一般获得实例的方式是使用构造函数，不过还有另外一种方式，一个类可以提供一个公共静态工厂方法，它只是一个返回类实例的静态方法。比如：

```java
public static Boolean valueOf(boolean b){
	return b ? Boolean.TRUE : Boolean.FALSE;
}
```

静态工厂方法的优点：

1. 静态工厂方法有名字。如果构造函数的参数本身并不能描述返回的对象，那么具有确切名称的静态工厂则更容易使用。例如，返回可能为素数的 BigInteger 类的构造函数 `BigInteger(int, int, Random)` 最好表示为名为 `BigInteger.probableprime` 的静态工厂方法。
2. 不需要在每次调用时创建新对象。
3. 可以获取返回类型的任何子类对象。
4. 返回对象的类可以随调用的不同而变化，作为输入参数的函数。
5. 当编写包含方法的类时，返回对象的类不需要存在。

静态工厂方法的缺点：

1. 类如果没有公共或受保护的构造器，就不能被子类继承。可以使用组合而不是继承。
2. 与其他静态方法没有区别，可以从名称上标识。

## Item 2:遇到多个构造器参数时，考虑使用Builder模式

一般对于一个构造器中有很多可选参数的情况，可以使用可伸缩的构造器，只是这样有很多参数的时候，会很难编写。

第二种是使用JavaBeans的方式，调用无参构造器创建一个对象，然后用setter方法来设置必要参数和可选参数。但是由于构造过程被分到了多个调用中，JavaBean在构造过程中可能处于不一致的状态。

第三种是Builder模式，客户端并不直接创建一个目标对象，而是先调用一个包含了所有必要参数的构造器（或静态工厂）进而得到一个builder对象。接着，客户端调用builder对象提供的类似于setter的方法，并根据喜好开始设置各个想可选参数。最后，客户端通过调用没有参数的build方法生成了目标对象，这个对象通常是不可变的。

不过在Kotlin中，有了参数默认值，就没有必要使用Builder模式。比如：

```kotlin
class KotlinNutritionFacts(
        private val servingSize: Int,
        private val servings: Int,
        private val calories: Int = 0,
        private val fat: Int = 0,
        private val sodium: Int = 0,
        private val carbohydrates: Int = 0)

val cocaCola = KotlinNutritionFacts(240,8,
                calories = 100,
                sodium = 35,
                carbohydrates = 27)
```

上面的代码中如果想要在创建完实例之后修改参数，可以把val变为var，去掉private或者在class中写函数改变参数的值。

如果想在Java中使用，可以通过kotlin的注解`@JvmOverloads`自动生成可伸缩的构造函数。

```kotlin
class KotlinNutritionFacts @JvmOverloads constructor(
        private val servingSize: Int,
        private val servings: Int,
        private val calories: Int = 0,
        private val fat: Int = 0,
        private val sodium: Int = 0,
        private val carbohydrates: Int = 0)
```
