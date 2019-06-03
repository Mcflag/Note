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

## Item 6:避免创建不必要的对象

很多时候，我们最好去复用一个对象而不是每次在需要时都去创建一个新的功能相同的对象。复用的方式不仅更快速，而且更时尚。如果一个对象是不可变的，那么它总是能被复用。

举一个极端的反面例子，考虑下面这个语句：

	String s = new String("bikini"); // DON'T DO THIS!

这条语句每次被执行时都会产生一个新的String实例，但这些对象的创建都是不必要的。改进的版本如下所示：

	String s = "bikini";

当不可变类同时提供了构造器和静态工厂方法时，我们优先使用静态工厂方法来避免创建不必要的对象。除了复用不可变的对象，我们也可以复用那些已知不会被修改的可变对象。

一些对象的创建的代价要比其它的对象要大。如果我们将不断地需要这些高代价对象，一个建议就是将其缓存起来并复用。不幸的是，什么时候需要创建这种对象并不总是那么明显。假设我们要编写一个方法来确定一个字符串是否是有效的罗马数字，最简单的方式就是通过使用正则表达式，如下所示：

```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})" + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$"); 
}
```

这种实现方式的问题在于它依赖于String.matches方法。虽然String.matches是校验一个字符串是否符合一个正则表达式最简单的方式，但在性能要求高的情景下，若想重复使用这种方式，就显得不是那么合适了。这里的问题是，它在内部创建了一个Pattern实例来用于正则表达式然后这个实例只使用一次，使用完之后它就被垃圾回收了。创建一个Pattern实例是昂贵的，因为它需要将正则表达式编译成一个有限状态机。

为了改善性能，我们可以将所需的正则表达式显式地编译进一个不可变的Pattern对象里，并让其作为类初始化的一部分，将其缓存起来。这样以后每次调用isRomanNumeral方法时，就可以复用相同的Pattern对象了：

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})"
                + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
    static boolean isRomanNumeral(String s) { 
        return ROMAN.matcher(s).matches();
    } 
}
```

如果包含了改进版的isRomanNumeral方法的类被初始化了，但这个方法缺从没被调用过，那么ROMAN属性的初始化就没什么意义了。虽然我们可以在isRomanNumeral方法被调用时才去初始化这个属性，从而减少了无意义的初始化，但这么做是不推荐的。延迟初始化（lazy initialization）会导致实现复杂化，而性能却没有可观的改进。

另一种会创建不必要对象的方式是自动装箱（autoboxing），这种方式允许程序员混用基本类型和装箱基本类型，然后按需自动装箱和解箱。自动装箱模糊了基本类型和装箱基本类型之间的区别，但并没有消除这种区别。我们一起来看看下面这个方法。这个方法计算了所有int正值的和。为了计算这个和，程序员必须采用long，因为一个int不够大，无法容纳所有int正值的总和：

```java
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;
    return sum; 
}
```

上述这段程序能获得正确的值，但它比实际情况要慢很多，因为程序里面打错了一个字符。变量sum被声明成了Long类型，而不是long，这意味着程序构造了2^31个不必要的Long实例（大约每次往Long类型的sum中增加long时构造一个Long实例）。将sum的类型从Long换成long时，在我的机器上运行时间从6.3秒减成了0.59秒。结论很明显：优先使用基本类型而不是装箱基本类型，同时小心无意识的自动装箱。

但千万不要将本条目误解为“创建对象的代价很昂贵，所以我们应该避免创建对象”。相反，那些构造器做很少显示工作的小对象的创建和回收是很廉价的，特别是在现代的JVM实现上。通过创建额外的对象来加强代码的清晰、简单或者功能，这通常是件好事。

相反，通过维护自己的对象池来避免创建对象也不是个好的实践，除非池中的对象是及其重量级的。真正正确使用对象池的经典例子是数据库连接池。由于建立数据库连接的代价是很高的，所以复用这些连接对象就显得很有意义了。然而，一般情况下，维护你自己的对象池会把代码弄的比较混乱，增加内存占用，而且还会降低性能。现代JVM实现具有高度优化的垃圾回收器，在对轻量级对象的处理上，这些回收器比对象池表现得更好。

