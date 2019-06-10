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

## Item 7:消除过时的对象引用

虽然Java有自动的垃圾回收机制，但是仍然需要考虑内存管理的问题。比如如下代码：

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    } 
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    } 
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    } 
    /**
     * Ensure space for at least one more element, roughly
     * doubling the capacity each time the array needs to grow.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

所以代码中内存泄露的部分在哪里？如果一个栈先增长后收缩，那些被弹出栈的对象将不会被回收，即使使用栈的程序不再引用它们。这是因为栈里面包含着这些对象的过期引用（obsolete reference）。过期引用是指永远不会被再次解除的引用。在这个例子当中，任何在elements数组的活动区域（active portion）之外的引用都是过期的。活动部分由elements数组里面下标小于数据长度的元素组成。

在支持垃圾回收的语言中，内存泄露的问题（更确切地说，是无意的对象保留）是很隐蔽的。如果一个对象的引用被无意保留了，不仅这个对象无法被回收，其它被这个对象引用的对象也无法被回收。即使只有少数几个对象引用被无意保留了，那么许许多多的对象也将跟着无法被回收，这里面潜伏着对性能重大的影响。

这类问题的解决办法很简单：一旦一个对象过时了，只需清空对象引用就可以了。

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

所以我们什么时候应该清空一个引用？

Stack类的哪个地方使得它有可能发生内存泄露？简单来说，因为它自己管理它自己的内存。存储池里都是elements数组里的元素（对象引用单元，而不是对象本身）。数组里活动区域的元素都是已分配的，数组其余部分的元素都是自由的。垃圾回收器无法知道这一点，因为对于垃圾回收器，elements数组里的所有对象引用都是等同的。只有程序员才知道数组非活动区域里的元素是不重要的。当某些元素变成非活动区域的一部分时，程序员可以立即手动将其清空。通过这种方式，程序员可以有效地告诉垃圾回收器可以回收了。

一般来说，每当一个类自己管理它的内存时，程序员就要小心内存泄露的问题了。无论什么时候，只要一个元素被释放了，这个元素包含的所有引用都应该清空。

缓存是内存泄露的另一个常见来源。当你往缓存里放了一个对象引用，就很容易忘记它还在那，使得它即使不再有用后还很长一段时间留在那里。有几个办法可以解决这个问题。如果你刚好要实现这么一个缓存，只要缓存之外存在只对某个key的引用，相应的项就有意义，那么就可以用WeakHashMap来充当这个缓存。但缓存中的项过期后，它们就会被自动清除。不过我们要记住，只有当所要的缓存项的生命周期是由该键的外部引用而不是值来决定时，WeakHashMap才有用。

第三种常见的缓存泄漏的来源是监听器和其它调用。比方说我们想实现一个API，客户端能在这个API中注册回调，但假如后面既没有显式地取消注册又没有采取某些行动，那么它们将逐渐积累。一种可以保证回调被及时垃圾回收的方式是，只保留对它们的弱引用。例如，我们可以将它们存储为WeakHashMap的键。由于内存泄漏通常不会表现为明显的失败，所以内存泄漏的问题可能在系统中存留好多年。它们往往只有可以通过仔细的代码检查或者在堆内存分析（heap profiler）工具的帮助下才能被发现。

## Item 8:避免使用终结方法和清理方法

终结方法（Finalizer）即finalize()方法，是不可预知的，很多时候是危险的，而且一般情况下是不必要的。使用它们会导致程序行为不稳定，性能降低还有移植问题。终结方法只有少数几种用途，我们将会本条目后面谈到。但根据经验，我们应该避免使用终结方法。比起终结方法，清理方法相对安全点，但仍是不可以预知的，运行慢的，而且一般情况下是不必要的。

终结方法和清理方法的一个缺点是无法保证它们及时地被执行。一个对象从变得不可到达开始到它的终结方法和清理方法被执行，中间可能会经过任意长的时间。这意味着，我们不应该在终结方法和清理方法中做对时间有严格要求的任务。例如，依赖终结方法或者清理方法来关闭文件资源是个严重的错误，因为打开文件的描述符是个有限的资源。如果在一段程序中很多文件都因为系统延迟执行终结方法或清理方法而停留在打开状态，那么当这段程序再打开一个文件时就会失败。

Java语言规范不仅不保证终结方法或清理方法会被及时运行，而且不保证它们最终会运行。这样的话完全有可能一个程序在终止的时候，某些已经无法访问的对象却还没被终结方法或清理方法处理。所以，我们应该永远也不依赖于终结方法或清理方法来更新持久化状态。例如，依赖于终结方法或清理方法来释放共享资源（比如数据库）的永久锁，将很容易使得整个分布式系统停止运行。

使用终结方法和清理方法还会导致严重的性能损失。换句话说，用终结方法来创建和销毁对象慢了大约50倍。这主要是因为终结方法会阻碍有效的垃圾回收。如果我们使用清理方法来清理类的所有对象，则其于终结方法速度相当，但如果我们将清理方法当作下面讨论到的安全网（safety net）来使用，则其比终结方法快很多。

终结方法还有一个严重的安全问题：它将你的类暴露于终结方法攻击（finalizer attack）。终结方法攻击的背后机制很简单：如果一个异常从构造器或者序列化中抛出，恶意子类的终结方法可以运行在本应夭折的只构造了部分的对象上。终结方法可以在一个静态属性上记录对象的应用，从而阻止这个对象被垃圾回收。一旦记录了有缺陷的对象，就可以简单地调用该对象上的任意方法，而这些方法本来就不应该允许存在。从构造方法里抛出异常应该足以防止对象被创建，但假如终结方法也存在，就不是这样了。这种攻击会带来可怕的后果。final类能免疫于此类攻击，因为没有人能对final类进行恶意继承。为了防止非final类遭受终结方法攻击，我们可以写一个什么都不做而且是final的终结方法。

所以，对于封装了需要终止使用的资源（比如文件或者线程），我们应该怎么做才能不用编写终止方法或者清理方法呢？我们只需让类继承AutoCloseable接口即可，并要求使用这个类的客户端在每个类实例都不再需要时就调用close方法，一般都是运用try-with-resources来保证资源的终止使用，即使抛出了异常，也能正确终止。这里有个细节值得提到，实例必须能对其是否被关闭保持追踪：close方法必须在一个属性里声明此对象不再有效，其它方法必须校验这个属性，如果对象被关闭后它们还被调用，就要抛出一个IllegalStateException异常。

那么清理方法或者终结方法有什么好处呢？它们可能有两种合法用途。

一种用途是作为安全网，以防资源拥有者忘了调用资源的close方法。虽然清理方法或者终结方法并不保证会被及时执行（或根本就不运行），但晚释放总比客户端忘了释放好。一些Java类库，如FileInputStream，FileOutputStream，ThreadPoolExecutor，还有java.sql.Connection，都有作为安全网的终结方法。

清理方法的第二种合法用途与对象的本地对等体（native peer）有关。本地对等体是指非Java实现的本地对象，普通对象通过本地方法代理给本地对象。由于本地对等体不是普通的对象，垃圾回收器并不知道它的存在进而当Java对等体被回收时也不会去回收它。而清理方法或终结方法正是适合完成这件事的工具，但前提条件是接受其性能并且本地对等体不持有关键资源。假如性能问题无法接受或者本地对等体持有的资源必须被及时回收，那么我们的类还是应该实现一个close方法，就如我们一开始提到。

清除器的使用有些棘手。下面是一个简单的 Room 类，展示了这个设施。让我们假设房间在回收之前必须被清理。Room 类实现了 AutoCloseable；它的自动清洗安全网使用了清除器，这只是一个实现细节。与终结器不同，清除器不会污染类的公共 API：

```java
// An autocloseable class using a cleaner as a safety net
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // Resource that requires cleaning. Must not refer to Room!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // Invoked by close method or cleaner
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // The state of this room, shared with our cleanable
    private final State state;
    // Our cleanable. Cleans the room when it’s eligible for gc
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
```

对 run方法的调用将由以下两种方法之一触发：通常是通过调用 Room 的 close 方法来触发，调用 Cleanable 的 clean 方法。如果当一个 Room 实例有资格进行垃圾收集时，客户端没有调用 close 方法，那么清除器将调用 State 的run 方法（希望如此）。状态实例不引用其 Room 实例是非常重要的。如果它这样做了，它将创建一个循环，以防止 Room 实例有资格进行垃圾收集（以及自动清理）。因此，状态必须是一个静态嵌套类，因为非静态嵌套类包含对其封闭实例的引用。

就像我们之前说的，Room 类的清除器只是用作安全网。如果客户端将所有 Room 实例包围在带有资源的 try 块中，则永远不需要自动清理。这位表现良好的客户端展示了这种做法：

```java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```

运行 Adult 程序打印Goodbye，然后是打扫房间。但这个从不打扫房间的不守规矩的程序怎么办？

```java
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
    }
}
```

你可能期望它打印出Peace out，然后打扫房间，但在我的机器上，它从不打扫房间；它只是退出。这就是我们之前提到的不可预测性。Cleaner 规范说：在 System.exit 中，清洁器的行为是特定于实现的。不保证清理操作是否被调用。虽然规范没有说明，但对于普通程序退出来说也是一样。在我的机器上，将 System.gc()添加到 Teenager 的主要方法中就足以让它在退出之前打扫房间，但不能保证在其他机器上看到相同的行为。总之，不要使用清洁器，或者在 Java 9 之前的版本中使用终结器，除非是作为安全网或终止非关键的本机资源。即便如此，也要小心不确定性和性能后果。

## Item 9:优先使用try-with-resources而不是try-finally

Java类库里包含了必须通过调用close方法来手动关闭的资源。比如InputStream，OutputStream还有java.sql.Connection。关闭资源这个动作通常被客户端忽视了，其性能表现也可想而知。虽然大部分这些资源都使用终结方法作为最后的安全线，但终结方法的效果并不是很好。在过去的实践当中，try-finally语句是保证一个资源被恰当关闭的最好的方式，即使是在程序抛出异常或者返回的情况下：

```java
// try-finally - No longer the best way to close resources!
static String firstLineOfFile(String path) throws IOException { 
    BufferedReader br = new BufferedReader(new FileReader(path)); 
    try {
        return br.readLine(); 
    } finally {
        br.close(); 
    }
}
```

这么做看起来可能还没什么问题，但当你添加第二个资源时，情况就开始变得糟糕了：

```java
// try-finally is ugly when used with more than one resource!
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src); 
    try {
        OutputStream out = new FileOutputStream(dst); 
        try {
            byte[] buf = new byte[BUFFER_SIZE]; 
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n); 
        } finally {
            out.close();
        }
    } finally {
        in.close(); 
    }
}
```

即使对于正确使用了try-finally语句的代码，如前面所示，也有个不起眼的缺点。无论是try里面的代码还是finally里面的代码，都有可能抛出异常。例如，在firstLineOfFile方法里，如果底层物理设备出了故障，则在调用readLine方法时会抛出异常，而且由于相同的原因，调用close方法也会失败。在这种情况下，第二种异常覆盖了第一种异常。在异常错误栈里将没有第一种异常的记录，这会使实际系统的调试变得很复杂，因为很多时候你是想查看第一种异常来诊断问题。虽然我们可以通过编写代码抑制第二种异常来让第一种异常显现出来，但几乎没人会这么做，因为这样的话代码就变得太冗长了。

当Java 7引入try-with-resources语句时，所有问题突然一下子解决了。若要使用这个语句，一个资源必须实现AutoCloseable接口，而这个接口只有一个返回类型为void的close（void-returning）方法。Java类库和第三方类库里面的许多类和接口现在都实现或继承了AutoCloseable接口。如果你写了一个类，这个类代表一个必须被关闭的资源，那么你的类也应该实现AutoCloseable接口。

```java
// try-with-resources - the the best way to close resources!
static String firstLineOfFile(String path) throws IOException { 
    try (
        BufferedReader br = new BufferedReader(new FileReader(path))
    ) { 
        return br.readLine();
    } 
}
```

这是我们使用try-with-resources的第二个例子：

```java
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
    try (
        InputStream in = new FileInputStream(src); 
        OutputStream out = new FileOutputStream(dst)
    ) {
        byte[] buf = new byte[BUFFER_SIZE]; int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n); 
    }
}
```

比起try-finally，try-with-resources语句不仅更简短和更可读，而且它们更容易排查问题。考虑firstLineOfFile方法的情况，如果从readLine方法和close（不可见）方法都抛出异常，那么后者抛出的异常将被抑制而不是前者。事实上，为了保留我们实际想看的异常，多个异常都可能会被抑制。这些被抑制的异常并不仅仅是被忽略了，它们被打印在错误栈当中，并被标注为被抑制了。

我们也可以像之前的try-finally语句那样，往try-with-resources里面添加catch子句。这能让我们无需在另一层嵌套污染代码就能处理异常。下面是一个比较刻意的例子，这个版本中的firstLineOfFile方法不会抛出异常，但如果它不能打开文件或者不能读打开的文件，它将返回一个默认值：

```java
// try-with-resources with a catch clause
static String firstLineOfFile(String path, String defaultVal) { 
    try (
        BufferedReader br = new BufferedReader(new FileReader(path))
    ) { 
        return br.readLine();
    } catch (IOException e) { 
        return defaultVal;
    } 
}
```

结论很明显：面对必须要关闭的资源，我们总是应该优先使用try-with-resources而不是try-finally。随之产生的代码更简短，更清晰，产生的异常对我们也更有用。try-with-resources语句让我们更容易编写必须要关闭的资源的代码，若采用try-finally则几乎做不到这点。