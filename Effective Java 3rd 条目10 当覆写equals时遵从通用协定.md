覆写equals方法看上去简单，但是有许多方式弄错这个问题，而且后果可能严重。避免这个问题的最容易的方式是，不要覆写equals方法，这种情况下，这个类的每个实例只等于它本身。如果下列条件之一符合，就是做对的事情：

 - **类的每个实例本质上是唯一的**。对于这些类，比如表示激活实体(active entity)而不是值的Thread，是正确的。对于这些类，Object提供的equals实现恰好有正确的行为。
 
 - **没有必要为类提供一个“逻辑上相等”的检测**。比如，java.util.regex.Pattern本可以覆写equals检测两个Pattern实例是否恰好表示同一个正则表达式，但是设计者认为客户端没必要或者不想有这个功能。在这些情况下，从Object继承的equals实现是合适的。
 
 -  **一个超类已经有覆写的equals，而且这个超类行为对于这个类来说是合适的**。比如，Set的大多数实现从AbstractSet继承它们的equals实现，List实现是从AbstractList，Map实现是从AbstractMap。
 
 -  **类是私有的或者包私有的，而且你肯定它的equals方法永远不会调用**。如果你是极端风险规避的，你可以覆写equals方法保证它不会意外调用：
```java
@Override public boolean equals(Object o) { 
	throw new AssertionError(); // 方法不会被调用 
}
```
那么什么时候覆写方法合适呢？*逻辑相等(logical equality)*不同于单单的对象标识，当类有这个概念，而且超类还没有覆写equals，这个时候是合适的。这通常是*值类(value class)*这种情况。一个值类仅仅是表示值的类，比如Integer或者String。程序员用equals方法比较值类的引用，期望找出它们是否在逻辑上相等，而不是它们是否引用同一个对象。覆写equals方法不仅仅对于符合程序员期望是必须的，而且它也使得实例作为map的键或者set的元素是可预测的和合理的行为。

不需要覆写equals方法的一种值类是这样的类，它使用实例控制(条目1)保证每个值最多存在一个对象。Enum类型(条目34)属于这一类。对于这些类，逻辑相等和对象标识是相同的，所以
Object的equals方法，它的作用相当于逻辑的equals方法。

当你覆写equals方法时，你必须遵守它的通用协定。下面是来自Object文档上的协定：

equals方法实现了*等价关系(equivalence relation)*。它有这些属性：

 - 反身性：对于任何非空的引用值x，x.equals(x)必须返回真。
 - 对称性：对于任何非空引用值x和y，当且仅当y.equals(x)返回真，x.equals(y)必须返回真。
 - 传递性：对于任何非空引用值x、y和z，如果x.equals(y)返回真，而且y.equals(z)返回真，那么x.equals(z)返回真。
 - 一致性：对于任何非空引用值x、y和z，x.equals(y)的多次调用必须一致返回真，或者一致返回假，假如用在equals比较上的信息没有改变。
 - 对于任何非空引用值x，x.equals(null)必须返回假。

除非你熟悉数学，这看上去可能有点吓人，但是不要忽略它。如果违反了它，你可能会发现，你的程序表现不稳定或者崩溃，有可能非常难于确定失败的来源。用John Donne的话说，类不是一个孤岛。一个类的实例经常传递给另外一个。许多类，包括所有的集合类，依赖于传入的对象，这些对象遵从于equals协定。

既然你已经知道违反equals协定的危险，那么让我们仔细检查这个协定。好消息是，尽管表面如此，它实际上不是特别复杂。一旦你理解了它，遵守它是不难的。

那么什么是等价关系呢？大约地讲，它是一个操作子，它把一个集的元素划分到子集，这个子集里面的元素被认为同另外一个是相等的。这些子集也称为相等类。对于一个有用的equals方法，每个相等类里面的所有元素，在用户的角度看，必须是可交换的。现在让我们依次检查这五项要求：

 - **反身性**：第一个要求说，一个对象必须和自身相等。很难想象无意违反了这个要求。如果你违反了它，然后把你的类的对象添加到一个集中，contains可能会说，这个集不含有你刚才添加的实例。
 
 - **对称性**：第二个要求说，任意两个对象必须对它们是否是相等的要达成一致。不像第一个要求，不难想象无意中违反这个要求。比如，考虑如下的类，它实现了大小写不敏感的字符串。这个字符串的大小写通过toString保存着，但是在equals比较中被忽略：

```java
// 已破坏 - 违反了对称性!
public final class CaseInsensitiveString { 
	private final String s;
	
	public CaseInsensitiveString(String s) { 
		this.s = Objects.requireNonNull(s); 
	}

	// 已破坏 - 违反了对称性!
	@Override public boolean equals(Object o) {
		if (o instanceof CaseInsensitiveString)
			return s.equalsIgnoreCase( 
				((CaseInsensitiveString) o).s);

		if (o instanceof String) // 单方向可交互!
			return s.equalsIgnoreCase((String) o);
		return false; 
	} 
	... // 其余省略
}
```

 这个类中出于好意的equals方法，徒劳地尝试和普通的字符串互操作。让我们假设，我们有一个大小写不敏感的字符串，还有一个普通的字符串：
```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish"); 
String s = "polish";
```
正如所料，cis.equals(s)返回真，问题在于，当CaseInsensitiveString的equals方法知道普通对象，但是String的equals方法不知道大小写不敏感的字符串。所以，s.equals(cis)返回假，一个明显的对称性的违反。假设你把大小写不敏感的字符串放入一个集：
```java
List<CaseInsensitiveString> list = new ArrayList<>(); 
list.add(cis);
```
这时候，list.contains(s)返回什么呢？谁知道呢？在当前的OpenJDK实现中，碰巧返回假，但那仅仅是一个人工实现。在另外一个实现中，它可能同样容易地返回真，或者抛出一个运行时错误。**如果你违反了equals协定，你简直不知道，其他对象面对你的对象时是什么行为**。

为了消除这个问题，仅需移除与String的equals方法交互这个错误的尝试。一旦这样做了，你可以重构这个方法到一个简单的返回语句：
```java
@Override public boolean equals(Object o) { 
	return o instanceof CaseInsensitiveString && 
		((CaseInsensitiveString) o).s.equalsIgnoreCase(s); 
}
```
 - **传递性**：equals协定的第三个要求说，如果一个对象和第二个相等，而且第二个对象和第三个相等，那么第一个对象必须和第三个相等。再次不难想象无意违反这个要求。考虑一个子类的情形，这个子类添加一个新值组件(value component)到它的超类。换句话说，子类添加一些信息，影响了equals比较。让我们以简单不可变的二维整形点类开始：
```java
public class Point {
	private final int x; 
	private final int y;

	public Point(int x, int y) { 
		this.x = x; 
		this.y = y; 
	}

	@Override public boolean equals(Object o) { 
		if (!(o instanceof Point)) 
			return false; 
		Point p = (Point)o; 
		return p.x == x && p.y == y; 
	}
	...// 其余省略
}
```
假设你想扩展这个类，给一个点添加颜色的概念：

```java
public class ColorPoint extends Point { 
	private final Color color;

	public ColorPoint(int x, int y, Color color) { 
		super(x, y); this.color = color; 
	}
	...// 其余省略
}
```
equals方法看起来怎么样呢？如果你完全忽视它，继承于Point的实现和颜色信息在equals比较时忽略了。虽然这不违反equals协定，但是明显是不可接受的。假设你写了一个equals方法，只要它的参数是另外一个相同位置和颜色的颜色点，就返回真：
```java
// 已破坏 - 违反对称性!
@Override public boolean equals(Object o) {
	if (!(o instanceof ColorPoint))
		return false;
	return super.equals(o) && ((ColorPoint) o).color == color; 
}
```
这个方法的问题是，当一个点和一个颜色点比较时，或者相反，可能有不同的结果。前者的比较忽略了颜色，而后者的比较总是返回假，因为参数的类型是不正确的。为了使这个更加具体，让我们创建一个点和一个颜色点：
```java
Point p = new Point(1, 2); 
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
```
于是，p.equals(cp)返回真，然而cp.equals(p)返回假。当进行“*混合比较*”是，你可能使用ColorPoint.equals时忽略颜色，来试着解决这个问题：
```java
// 已破坏 - 违反传递性! 
@Override public boolean equals(Object o) {
	if (!(o instanceof Point))
		return false;

	// 如果o是个通常的Point，进行无关颜色的比较 
	if (!(o instanceof ColorPoint)) 
		return o.equals(this);

	// o是ColorPoint类型; 进行全面的比较 
	return super.equals(o) && ((ColorPoint) o).color == color;
}
```
这个方法提供了对称性，但是是以传递性为代价：
```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED); 
Point p2 = new Point(1, 2); 
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```
现在p1.equals(p2)和p2.equals(p3)返回真，而p1.equals(p3)返回假，这个明显的传递性违反。前面两个比较是无关颜色的，而第三个考虑了颜色。

而且，这个方法可能造成无限递归：假设有两个Point子类，比如说ColorPoint和SmellPoint，每个子类都有这样的equals方法。然后myColorPoint.equals(mySmellPoint)的调用将抛出StackOverflowError。

那么解决方案是什么呢？结果是，在面向对象语言中，这是一个等价关系的基本问题。**当维护equals协定时，没办法扩展一个不可实例化的类和添加一个值组件**，除非你愿意放弃面向对象的抽象性。

你可能听说过，可以扩展一个不可实例化的类和添加一个值组件，而且以getClass检测代替instanceof检测的方式维护equals协定：
```java
// 已破坏 - 违反了里氏代换原则
@Override public boolean equals(Object o) { 
	if (o == null || o.getClass() != getClass()) 
		return false; 
		
	Point p = (Point) o; 
	return p.x == x && p.y == y; 
}
```
只有在它们有相同实现类的时候，才有相等对象的效果。这可能不算太糟，但是结果是不可接受的：Point子类的实例仍然一个Point，而且仍然需要起到它的作用，但是， 如果你使用这个方法，它不能够起这样的作用。假设我们想要编写一个方法，辨别一个点是否在一个单元圆上。下面是我们可以这么做的一种方式：
```java
// 初始化单位元，包含单元圆上的所有点
private static final Set<Point> unitCircle = Set.of(
		new Point( 1, 0), new Point( 0, 1), 
		new Point(-1, 0), new Point( 0, -1));

public static boolean onUnitCircle(Point p) { 
	return unitCircle.contains(p); 
}
```
虽然这可能不是最快的方式实现这个功能，但是它工作良好。假设你用某种简单的方式扩展Point，而不会添加一个值组件，即，通过它的构造子跟踪创建了多少个实例：
```java
public class CounterPoint extends Point {
	private static final AtomicInteger counter = new AtomicInteger();
	
	public CounterPoint(int x, int y) { 
		super(x, y); 
		counter.incrementAndGet(); 
	} 
	public static int numberCreated() { 
		return counter.get(); 
	}
}
```
里氏代换原则说，一个类型的任何重要属性也应该对所有它的子类有效，以便为类型编写的任意方法，同样应该在它的子类上工作良好 [Liskov87]。这是我们更早声称的正式说明，这个声称说，Point子类，比如CounterPoint，仍然是一个Point，而且必须充当它的作用。但是，假设我们传递CounterPoint到onUnitCircle。如果Point类使用基于getClass的equals方法，那么onUnitCircle方法将返回假，不管CounterPoint实例的x和y坐标。正是如此，因为大多数数据集，包括被CounterPoint使用的HashSet，用equals方法来检测包含，而且CounterPoint实例不等于任何Point。然而，如果你使用恰当的基于instanceof的Point的equals方法，那么当面对CounterPoint实例时，同一个onUnitCircle方法工作良好。

虽然扩展不可实例化的类和添加值组件是不令人满意的方式，但是有个好的变通方案：根据条目18(组合优于继承)的建议，不是让ColorPoint扩展Point，而是给ColorPoint一个私有的Point域和一个公开的*视图方法(view method)*(条目6)，这个方法返回像这个颜色点同样位置的一个点：

```java
// 添加一个值组件，而没有违反equals协定 
public class ColorPoint {
	private final Point point; 
	private final Color color;

	public ColorPoint(int x, int y, Color color) { 
		point = new Point(x, y); 
		this.color = Objects.requireNonNull(color); 
	}

	/** 
	  * 返回这个颜色点的Point的视图 
	  */ 
	public Point asPoint() { 
		return point; 
	}

	@Override public boolean equals(Object o) { 
		if (!(o instanceof ColorPoint)) 
			return false; 
		ColorPoint cp = (ColorPoint) o; 
		return cp.point.equals(point) && cp.color.equals(color); 
	}
	...// 其余省略
}
```
在Java平台库里有些类确实扩展一个不可实例化的类，而且添加了一个值类型。比如，java.sql.Timestamp扩展了java.util.Date，而且添加了nanoseconds域。Timestamp的equals实现确实违反了对称性而且可能造成不稳定的行为，如果Timestamp和Date对象在同一个数据集中或者另外的混用。只要你把它们分开，你就不会遇到麻烦，但是没有任何事情可以阻止你混用它们，而且造成的错误很难调试。Timestamp类的行为是一个错误，不应该模仿。

注意，你可以添加值组件到抽象类的一个子类中，而没有违反equals协定。这是重要的，对于根据在条目23(类层级优于标记类)的建议，你获取到这种类层级。比如你可以有个没有值组件的抽象Shape类，Circle子类添加了一个radius域，而且Rectangle子类添加了长和宽的域。只要不可能直接创建一个超类实例，前面展示的这种问题不会发生。

 - 一致性：equals协定的第四个要求是，如果两个对象是相等的，那么它们所有时间都必须相等，除非其中之一或者两者被修改了。换句话说，可变对象在不同的时间可能等于不同的对象，然而不可变对象是不能的。当你编写一个类，认真思考它是否应该是不可变的(条目17)。如果你断定它应该，确保你的equals方法限制了相等对象永远相等，不相等对象永远不相等。

不管一个对象是否可变，**不要编写一个依赖不可靠资源的equals方法**。如果你违反了这个禁令，满足一致性是极其困难的。比如，java.net.URL的equals方法依赖于与主机IP地址有关系的URL的比较。翻译主机名到IP地址，可能需要网络连接，而且不能够保证随着时间变化而产生相同的结果。这个可能造成URL的equals方法违反了equals协定，而且在实际中引发问题。URL的equals方法的行为是一个大的错误，不应该模仿。不幸的是，由于兼容性要求，这不能够改变。为了避免这样的问题，equals方法应该仅仅对驻留内存的对象，执行确定性的计算。

- 非空性： 最后一个要求缺少官方名字，所以我冒昧地叫它“非空性”。它是说，所有的对象应该不与空相等。虽然难于想象，对o.equals(null)的调用的响应，意外返回真，但是不难想象意外抛出一个NullPointerException。这个通用协定禁止这个。许多类有equals方法，用一个显式的null检测防止它：
```java
@Override public boolean equals(Object o) { 
	if (o == null) 
		return false; 
	...
}
```
这个检测是必要的。为了相等检测它的参数，equals方法必须首先强转它的参数到一个正确的类型，使得它的访问器可以被调用或者它的域可以被访问。在进行强转之前，方法必须使用instanceof操作子，检查它的参数是正确的类型：
```java
@Override public boolean equals(Object o) { 
	if (!(o instanceof MyType)) 
		return false; 
	MyType mt = (MyType) o;
	...
}
```
如果这个类型检查缺少，而且equals方法传入了一个错误的参数，equals方法应该抛出ClassCastException，这违反了equals协定。但是instanceof操作子指定返回假，如果他的第一个操作数是空，不管第二个操作数的类型是什么[JLS, 15.20.2]。所以，如果类型检查返回假，所以你不需要显式的空检查。

综合前述，下面是一个高质量的equals方法的秘诀：

 1. **使用==操作子检测参数是否是这个对象的一个引用**。如果如此，返回真。这只是一个性能优化，但是如果对比可能代价大时，这件事情是值得做的。
	 
 2. **使用instanceof操作子检查参数是否有相同的类型**。如果没有，返回假。通常，正确的类型是一个类，这个方法在这个类里面出现。有时，类型是这个类实现的某个接口。如果类实现了一个接口，这个接口改善了equals协定，允许实现该接口的类之间比较，那么使用接口。Collection接口，比如Set、List、Map和Map.Entry，有这个属性。
 
 3.  **强转参数到正确的类型**。因为这个强转之前有instanceof检测，保证了成功。
 
 4.  **在类里面的每个“重要”域，检测参数的域是否匹配这个对象的相应的域**。如果所有这些检测通过，返回真；否则，返回假。如果步骤2的类型是一个接口，你必须通过接口的方法获取参数的域；如果类型是类的话，你或许可能直接获取域，决定于它们的可存取性。

对于原始的域，而且它的类型不是float或者double，使用==操作子比较；对于对象引用的域，递归地调用equals方法；对于float域，使用静态的Float.compare(float, float)；对于double域，使用Double.compare(double, double)。float和double域的特别处理是必要的，由于Float.NaN, -0.0f和double类似这样的值。详细情况参考JLS 15.21.1或者Float.equals的文档。虽然你可以用静态方法Float.equals和Double.equals对比float和double域，但是每次比较，会需要自动装箱，这使得有糟糕的性能。对于队列的域，使用这些规则到每个元素。如果队列域中的每个元素都重要，使用Arrays.equals方法之一。

一些对象引用域，可能合理地包括空。为了避免NullPointerException的可能性，使用静态方法Objects.equals(Object, Object)，为相等性检测这些域。

对于一些类，比如上面的CaseInsensitiveString，域比较比简单的相等检测复杂的多。如果是这个情况，你可能想要存储域的*标准形式(canonical form)*，这样，equals方法可以用标准形式进行便宜精确的比较，而不是一个代价更高的非标准的比较。这个技巧对不可变类是最合适的(条目17)；如果这个对象可以改变，你必须保持标准形式最新。

equals方法的性能可能会被它里面域的比较顺序所影响。为了最好的性能，你应该首先比较这样的域：更可能不同的、对比代价更低的，或者理想地，两者。你不可以比较不是对象逻辑状态的部分的域，比如用在synchronize操作上的锁域。你不需要比较派生的域，这个可以从“重要的域”计算出来，但是这样做可以改进equals方法的性能。如果派生的域相当于整个对象概要描述，比较失败时，比较这个域将为你省下比较实际数据的代价。举个例子，假设你有一个Polygon类，而且你缓存了这个地方。如果两个多边形有不同的地方，你不需要麻烦比较它们的边和顶点。

**当你完成编写equals方法，问你自己这个三个问题：它是对称的吗？它是可传递的吗？它是一致的吗？**不要仅仅问自己；编写单元测试来检测，除非你使用AutoValue生成你的equals方法，这种情况下，你可以安全地省略检测。如果属性不能够，那么弄清楚为什么，然后相应修改equals方法。当然，你的equals方法也必须满足另外两个属性(反身性和非空性)，但是这两个属性通常由它们自己处理。

根据前面的秘籍构建的equals方法，用这个简化的PhoneNumber类显示：
```java
// 典型equals方法的类
public final class PhoneNumber { 
	private final short areaCode, prefix, lineNum;

	public PhoneNumber(int areaCode, int prefix, int lineNum) { 
		this.areaCode = rangeCheck(areaCode, 999, "area code"); 
		this.prefix = rangeCheck(prefix, 999, "prefix"); 
		this.lineNum = rangeCheck(lineNum, 9999, "line num"); 
	}

	private static short rangeCheck(int val, int max, String arg) { 
		if (val < 0 || val > max) 
			throw new IllegalArgumentException(arg + ": " + val); 
		return (short) val; 
	}

	@Override public boolean equals(Object o) {
		if (o == this) 
			return true; 
		if (!(o instanceof PhoneNumber))
			return false; 
		PhoneNumber pn = (PhoneNumber)o; 
		return pn.lineNum == lineNum && pn.prefix == prefix
			&& pn.areaCode == areaCode;
	} 
	... // 其余省略
}
```
下面是一些最后的警告：

- **当你覆写equals时，永远覆写hashCode**(条目11)

- **不要自作聪明**。如果你简单地为相等性检测域，遵守equals协定是不难的。如果你过度积极寻找等价，容易陷入麻烦。考虑任何混叠，通常只是一个坏主意。比如，File类不应该尝试使同个文件的符号链接相等。幸好，它没有。

- **在equals声明中，不要为Object替代另外的类别**。对于一个程序员，这不是不常见的，编写一个equals方法，看上去这样，然后花几个小时苦苦考虑为什么没有正常工作。

```
// 已破坏 - 参数类型必须是Object! 
public boolean equals(MyClass o) { 
	...
}
```
问题是，这个方法没有覆写Object.equals，它的参数是Object类别，而是相反，重载了它(条目52)。这是不可接受的，提供这样一个“强类型”的equals方法，甚至在正常的方法之外，因为它造成子类中的Override注解生成一个错误的正值，而且提供了一个安全的错觉。

Override注解的一致使用，就像这个条目中一直阐述的，将阻止你犯这个错误(条目40)。这个equals方法编译不通过，而且这个错误信息将告诉什么问题。
```java
// 仍然已破坏，但是编译不通过 
@Override public boolean equals(MyClass o) { 
	...
}
```
编写和测试equals(和hashCode)方法是乏味的，而且产生的结果是无奇的。手动编写和测试这些方法，有个绝佳的替代是使用谷歌的AutoValue开源框架，它会自动为你生成这些方法，由类的注释触发。在大多数情况，AutoValue生成的方法和你自己编写的方法本质上是相同的。

同样，IDE有工具生成equals和hashcode方法，但是产生的源代码，相对于使用AutoValue，更加冗长和更不可读，不要在类中自动地跟踪改变，所以需要测试。这就是说，用IDE生成equals(和hashCode)方法，相对于手动实现他们，通常是更加可取的，因为IDE不会犯粗心大意的错误，但是人类会。

总之，不要覆写equals方法，除非你不得不：在许多情形下，从Object继承的实现恰好是如你所想的。如果你确实要覆写equals，确保比较所有类的重要域，而且以一种符合equals协定的五种条款的方式比较它们。
