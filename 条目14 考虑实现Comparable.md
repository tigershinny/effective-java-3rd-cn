不像本章讨论的其他方法，compareTo方法不是在Object中声明的。更确切地说，它是Comparable接口的唯一方法。在角色上像Object的equals方法，除了它允许简单的相等比较之外，它还允许顺序比较，而且它是支持泛型的。通过实现Comparable，一个类表明它的实例有一个*自然排序(natural ordering)*。由实现Comparable的对象组成的一个队列，它的排序就像下面一样简单：
```java
Arrays.sort(a);
```
查找、计算极端值和维护自动排序的Comparable对象数据集，同样简单。比如，如下的程序，依赖于String实现了Comparable这个事实，打印了一个字母列表，去除了它的重复命令行参数：
```java
public class WordList { 
	public static void main(String[] args) { 
		Set<String> s = new TreeSet<>(); 
		Collections.addAll(s, args); 
		System.out.println(s); 
	} 
}
```
通过实现Comparable，你允许你的类与之相互操作：所有的众多泛型算法和依赖于这个接口的数据集实现。你以小量的努力获得到巨大的能力。事实上，Java平台库中的所有值类和所有枚举类型(条目34)，实现了Comparable。如果你编写明显自然排序的一个值类，比如字母排序、数值排序或者按年代排序，那么你应该实现Comparable接口：
```java
public interface Comparable<T> { 
	int compareTo(T t); 
}
```
compareTo的通用协定是同equals相似的：

> 把指定对象和这个对象依顺序比较。当这个类小于、等于或者大于指定对象时，返回一个负整数、零或者一个正整数。如果指定对象的类别阻止了它同这个对象比较，抛出ClassCastException。
> 
> 在下面的描述中，把符号sgn(*expression*)定名为数学上的signum函数，它根据表达式的值是否为负、零或者正，定义为返回-1、0或者1。
> 
> - 对于所有的x和y，实现者必须保证sgn(x.compareTo(y)) == -sgn(y.compareTo(x))。(这意味着，如果而且只有y.compareTo(x)抛出一个异常，x.compareTo(y)必须抛出一个异常。)
> 
> - 实现者也必须保证这个关系是可传递的：(x.compareTo(y) > 0 && y.compareTo(z) > 0)意味着x.compareTo(z) > 0。
> 
> - 最后，对于所有的z，实现者必须保证x.compareTo(y) == 0意味着sgn(x.compareTo(z)) == sgn(y.compareTo(z))
> 
> -  (x.compareTo(y) == 0) == (x.equals(y))，是强烈建议的，但是不是必要的。通常地讲，实现了Comparable接口而且违反了这个条件的任何类，应该清楚地表明这个事实。推荐的语言是：“注意：这个类有一个自然排序，与equals不一致。”

不要让这个协定的数学特质令你退却了。就像equals协定(条目10)，这个协定不是像看上去那么复杂。不像equals方法强制所有对象上的整体相等关系，compareTo不需要在不同类型对象之间起作用：当面对不同类型的对象，compareTo允许抛出ClassCastException。通常，这确切是它应该做的。这个协定确实允许类型间比较，它通常在一个对比对象实现的接口中定义。

就像违反了hashCode协定的类可能破坏依赖于哈希的其他类，违反compareTo协定的类可能破坏依赖于比较的其他类。依赖于比较的类包括，排序数据集TreeSet和TreeMap，和保函了查找和排序算法的效用类Collections和Arrays。

让我们重温compareTo协定的条款。第一个条款说，如果你反转两个对象引用之间比较的顺序，那么期待这样的事情发生：如果第一个对象小于第二个，那么第二个必须大于第一个；如果第一个对象等于第二个，那么第二个必须等于第一个；如果第一个对象大于第二个，那么第二个必须小于第一个。第二个条款说，如果一个对象大于第二个，而且第二个大于第三个，那么第一个必须大于第三个。最后一个条款说，比较为相等的所有对象，当与任何其他对象相比较时，必须产生相同结果。

这三个条款的结果是，compareTo方法强加的相等检测必须遵从equals协定强加的相同限制：反身性、对称性和传递性。所以，相同的警告也适用：用新的值组件扩展一个不可实例化的类，而且维护compareTo协定，这是不可能的，除非你愿意放弃面向对象的抽象的益处(条目10)。同样的变通方法也适用。如果你想添加一个值组件到一个实现Comparable的类，那么不要扩展它；编写一个不相关的类，它包含第一个类的实例。然后提供了一个“视图”方法，返回被包含的实例。这让你解脱出来，在包含的类上实现你想的compareTo方法任何东西，而且当需要的时候，让客户端把包含类的实例看作被包含类的实例。

compareTo协定的最后一个段落，它是强烈建议而非一个真正的必要条件，简单地陈述为，compareTo强加的相等检测应该通常与equals方法一样返回同样的结果。如果遵从这个条款，这个由compareTo强加的排序被认为与equals*一致*。如果违反了它，那么排序被认为与equals*不一致*。一个类的compareTo方法强加一个与equals*不一致*的顺序，这个类仍然起作用，但是包含类元素的排序数据集可能不会遵从恰当数据集接口(Collection、Set或者Map)的通用协定。这是因为这些接口的通用协定是依据equals方法定义的，但是排序数据集使用compareTo(而不是equals)强加的相等检测。如果这个发生了，这不会是一个灾难，但是要意识到这个事情。

比如，考虑BigDecimal类，它的compareTo方法与equals是不一致的。如果你创建了一个空HashSet实例，然后添加new BigDecimal("1.0") 和new BigDecimal("1.00")，那么这个集将包含两个元素，因为当使用equals方法比较时，添加到集的这两个BigDecimal实例是不相等的。然而，如果你使用TreeSet而不是HashSet执行相同的过程，这个集将仅仅包含一个元素，因为当使用compareTo方法比较时，这两个BigDecimal实例是相等的。(细节参考BigDecimal文档。)

编写compareTo方法与编写equals方法是相似的，但是有一些关键的区别。因为Comparable接口是参数化的，compareTo方式是静态类型的，所以你没必要类型检查或者强转它的参数。如果参数是错误的类型，那么调用甚至不会通过编译。如果参数是空，那么调用将会抛出NullPointerException，而且一旦方法尝试获取它的成员它将抛出。

compareTo方法里面，域是为顺序而不是相等而比较的。为了比较对象引用的域，递归地调用compareTo方法。如果一个域没有实现Comparable，或者你需要一个非标准的排序，那么改为使用Comparator。你可以编写你自己的comparator，或者使用一个存在的comparator，就像在条目10中CaseInsensitiveString的下面这个compareTo方法：

```java
// 对象引用域的单个域Comparable
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> { 
	public int compareTo(CaseInsensitiveString cis) { 
		return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s); 
	} 
	... // 其余省略 
}
```
注意，CaseInsensitiveString实现了Comparable&lt;CaseInsensitiveString&gt;。这意味着，CaseInsensitiveString引用仅仅可以同其他CaseInsensitiveString引用相比较。当声明一个类实现Comparable时，这是要遵循的标准模式。

这本书前期版本推荐说，compareTo方法使用关系操作子<和>比较整数原始域，使用静态方法Double.compare和Float.compare比较浮点数原始域。在Java 7中，静态compare方法加入到了所有Java原始装箱类。**使用compareTo方法的关系操作子<和>是冗余的、容易出错的和不再推荐的**。

如果一个类有多个重要域，比较它们的顺序是重要的。从最重要的域开始，然后按照这种方式往下做。如果比较结果不是零(零代表相等)，比较结束，仅仅返回这个结构。如果最重要的域是相等的，比较次最重要的域，等等，直至你发现了一个不相等的域或者比较最不重要的域。以下是是条目11 PhoneNumber类的compareTo方法，展示这个技巧：

```java
// 原始域的多域比较 
public int compareTo(PhoneNumber pn) {
	int result = Short.compare(areaCode, pn.areaCode); 
	if (result == 0) {
		result = Short.compare(prefix, pn.prefix);
		if (result == 0)
			result = Short.compare(lineNum, pn.lineNum); 
	} 
	return result;
}
```
在Java 8中，Comparator接口配备了*比较子构建方法(comparator construction methods)*集，它们使得比较子流畅构建。这些比较子然后可以用作实现一个compareTo方法，就像Comparable接口要求的。许多程序员更喜欢这个方法的简洁，虽然它确实是以一定性能代价实现的：PhoneNumber实例队列的排序在我的机器上慢10%。当使用这个方法时，考虑使用Java的*静态导入(static import)*特性，因此为了清晰和简洁，你应该通过它们的简短名字，引用静态比较子构建方法。使用这个方法，以下是PhoneNumber的compareTo方法的样子：
```java
// 比较子构建方法的Comparable
private static final Comparator<PhoneNumber> COMPARATOR = 
	comparingInt((PhoneNumber pn) -> pn.areaCode) 
		.thenComparingInt(pn -> pn.prefix) 
		.thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) { 
	return COMPARATOR.compare(this, pn);
}
```
这个实现在类初始化的时候建造了一个比较子，使用两个比较子构建方法。第一个是comparingInt。它是一个静态方法，接受一个*键抽取器函数(key extractor function)*，这个函数把对象引用映射到int类型的键，然后返回根据那个键排序对象的一个比较子。在前面的例子中，comparingInt接受lambda ()，它从PhoneNumber抽取地区号码，然后返回一个Comparator&lt;PhoneNumber&gt;，它根据它们的地区号码对电话号码排序。注意，lambda显式地指定了它的输入参数类型(PhoneNumber pn)。在这种情况下，结果是，Java的类型推导不足够强大到它自己能弄明白这个类型，所以为了让程序编译成功，我们不得不帮助它。

如果两个电话号码有相同的地区号码，我们需要进一步优化这个比较，那正是第二个比较子构建方法thenComparingInt所做的。它是Comparator上的一个实例方法，接受一个int键抽取器函数，而且返回一个比较子，它首先应用于原来的比较子然后使用抽取的键来来隔断关系。你可以叠加你想要任意多的thenComparingInt调用，这导致了*字典式排列(lexicographic ordering)*。在上面的例子中，我们叠加了两个thenComparingInt调用，最终是这样的排序，它的第二个键是前缀而且它的第三个键是线路号码。注意到，我们没有必要指定传入到thenComparingInt调用之一的键抽取器函数的参数类型：Java类型推导足够聪明到它自己弄明白这个类型。

Comparator类有一个全面的构建方法。对于原始类型long和double，有类似于comparingInt和thenComparingInt。int版本也可以使用在更加窄的整数类型，比如short，就像在我们的PhoneNumber例子。double类型也可以使用为float。这提供了所有Java原始数值类型的覆盖。

而且有为对象引用类型的comparator构建方法。这个叫comparing的静态方法，有两个重载。一个接受键抽取器，使用键的自然排序。第二个接受一个键抽取器和一个使用在抽取键上的比较子。有叫thenComparing这个实例方法的三个重载。一个重载仅仅接受一个比较子，使用它次级排序。第二个重载仅仅接受一个键抽取器，使用键的自然排序作为次级顺序。最后一个重载同时接受键抽取器和使用在键抽取键上的比较子。

偶尔你可以看见compareTo或者compare方法依赖于这个事实：如果第一个值小于第二个，两个值之间的差是负的，如果两个值相等，为零，如果第一值比较大，为正。下面是一个例子：

```java
// 已破坏 基于差的比较子 - 违反了传递性! 
static Comparator<Object> hashCodeOrder = new Comparator<>() {
	public int compare(Object o1, Object o2) {
		return o1.hashCode() - o2.hashCode();
	} 
};
```
不要使用这个技巧。它充满危险的：整数溢出和IEEE 754浮点数算术考究[JLS 15.20.1, 15.21.1]。再者，最后的方法不可能比使用这个条目描述的技巧编写的那些方法更快。要么使用这个静态compare方法：

```java
// 基于静态compare方法的Comparator
static Comparator<Object> hashCodeOrder = new Comparator<>() {
	public int compare(Object o1, Object o2) {
		return Integer.compare(o1.hashCode(), o2.hashCode());
	} 
};
```
要么使用comparator构建方法：
```
// 基于Comparator构建方法的Comparator
static Comparator<Object> hashCodeOrder = 
	Comparator.comparingInt(o -> o.hashCode());
```
总之，每当你实现了一个排序敏感的值类，你应该让这个类实现Comparable，以便它的实例可以容易地排序、查找和在基于比较的数据集中使用。当在compareTo方法实现中比较域值时，避免使用<和> 操作子。相反，在原始装箱类中，或者在Comparator接口中的比较子构建方法中，使用静态compare方法。
