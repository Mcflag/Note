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
## Item 3:使用私有构造器或者枚举类型来强化Singleton属性

Singleton就是单例，指仅被实例化一次的类。Singleton通常被用来表示一个无状态的对象，比如函数，或者一个独一无二的系统组建。

```java
public class JavaElvis {

    private static JavaElvis instance;

    private JavaElvis() {}

    public static JavaElvis getInstance() {
        if (instance == null) {
            instance = new JavaElvis();
        }
        return instance;
    }

    public void leaveTheBuilding() {
    }
}
```

另外还可以声明一个包含单个元素的枚举来实现单例。如果我们的Singleton必须扩展自一个超类而不是枚举时，这种方式就不能使用了（虽然你也可以申请一个枚举，这个枚举实现自一个或多个接口）

```java
public enum Elvis { 
    INSTANCE;
    public void leaveTheBuilding() { ... } 
}
```

在kotlin中直接使用object就可以实现一个单例。

```kotlin
object KotlinElvis {
    fun leaveTheBuilding() {}
}
```

## Item 4:通过私有化构造器强化不可实例化的能力

通过将类做成抽象类以试图强制不可实例化是行不通的。因为抽象类能被继承而且相应的子类能被实例化。不仅如此，将类抽象化也会误导用户，让用户以为这个类是被设计成来继承的。

将构造器私有化让其不可被实例化。不过这样做的副作用是该类不能被继承。

```java
public class UtilityClass {
    private UtilityClass() {
        throw new AssertionError();
    }
}
```

## Item 5:优先使用依赖注入而不是硬连接资源

很多类都依赖一个或多个底层资源。例如，拼写检查器依赖于字典。我们常常会见到这种类被做成了静态工具类，我们也会常常见到这种类被做成了Singleton。

静态工具类和Singleton对于类行为需要被底层资源参数化的场景是不适用的。

比如有一个拼写检查的类SpellChecker，用到了一个字典资源dictionary，但是字典可能有多种。我们需要的是能支持多个实例的类（例子中对应的是SpellChecker），每个实例使用对应客户端需要用到的资源（例子中对应的是dictionary）。

在创建一个新的实例时将资源参数传入构造器。这是依赖注入（dependency injection）的一种形式：词典作为拼写检查器的依赖在检查器被创建时就被注入了。

```java
public class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    } 
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

依赖注入不仅适用于构造器，也同样适用于静态工厂和builder。

这种模式的一个有用的变体是，往构造器里传入一个资源工厂。这个工厂是一个能被重复调用并产生某个类型实例的对象。

这些工厂是工厂方法模式（Factory Method pattern）的体现。Java 8中引入的Supplier<T>接口就能很好地表示这些工厂。对于将Supplier<T>作为输入的方法，我们通常应该使用有界通配符类型（bounded wildcard type）来限制工厂的参数类型，以允许客户端能传入一个工厂，而这个工厂能创建指定类型任意子类型实例。

```java
Mosaic create(Supplier<? extends Tile> tileFactory) { ... }
```

总的说来，对于需要依赖一个或多个底层资源的类，而且这些资源的行为还会影响到类时，我们不要用Singleton或者静态工具类类实现，也不要让类自己直接去创建这些资源，而是应该将这些资源或者创建资源的工厂传入构造器（或者静态工厂和builder）。