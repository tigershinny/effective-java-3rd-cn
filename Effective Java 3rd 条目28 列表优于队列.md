队列在两个方向上与泛型不相同。首先，队列是协变的。这个听上去令人不安的单词意思仅仅是：如果Sub是Super的一个子类，那么队列类型Sub[]是Super[]的一个子类。相反，泛型是非协变的：对于任何两个不同类型Type1和Type2，List&lt;Type1&gt;既不是List&lt;Type2&gt;的一个子类，也不是它的超类[JLS, 4.10; Naftalin07, 2.5]。你可能认为这意味着，泛型是有缺陷的，然而大概是队列有缺陷的。以下代码片段是合法的：
```java
// 运行时失败!
Object[] objectArray = new Long[1]; 
objectArray[0] = "I don't fit in"; // 抛出ArrayStoreException
```
但是如下不是合法的：
```java
// 编译不通过!

List<Object> ol = new ArrayList<Long>(); // 不可兼容类型 
ol.add("I don't fit in");
```
两种方式都不能把String放到Long容器里面，但是使用队列，你会在运行时发现一个错误；使用列表，你会在编译时发现一个错误。当然，你应该宁愿在编译时发现错误。
队列和泛型之间第二个主要区别是，队列是具体化的[JLS, 4.7]。这意味着，队列在运行时知道和约束了它们的元素类型。就像以前提到的，如果你试着把一个String到一个Long队列中，你将得到ArrayStoreException。相反，泛型是实施为类型擦除的[JLS, 4.6]。这意味着，它们仅仅在编译时约束了它们的类型，而且在允许时抛弃了(擦除了)它们的元素类型信息。擦除使得泛型类型和不使用泛型的遗留代码自由地互操作(条目26)，这保证了在Java5中顺利地过度到泛型。

因为这些基本的不同，队列和泛型不能够很好地混合使用。例如，创建一个泛型类型的、参数化类型的或者类型参数的队列是不合法的。所以，这些队列创建表达式都不是合法的：new List&lt;E&gt;[], new List&lt;String&gt;[], new E[]。所有这些都会导致编译时泛型队列创建错误。

创建泛型队列为什么是不合法的呢？因为它不是类型安全的。如果它是合法的，在其他正确程序中，编译器产生的强转可能在运行时以ClassCastException方式失败了。这违反了泛型类型系统提供的基本保证。

为了使得更加具体，考虑如下代码片段：
```java
// 泛型队列创建为什么是不合法的 - 编译不通过! 
List<String>[] stringLists = new List<String>[1]; // (1) 
List<Integer> intList = List.of(42); // (2) 
Object[] objects = stringLists; // (3) 
objects[0] = intList; // (4) 
String s = stringLists[0].get(0); // (5)
```
让我们假设，第一行，创建了泛型队列，是合法的。第二行创建和初始化了一个List&lt;Integer&gt;，它包含了单一元素。第三行保存List&lt;String&gt;队列到Object队列变量中，这是合法的，因为队列是协变的。第四行保存 List&lt;Integer&gt; 到Object队列的单个元素中，这是可以成功的，因为泛型是实现擦除的：List&lt;Integer&gt;实例的运行类型仅仅是List，而且List&lt;String&gt;[]实例的运行类型是List[]，所以这个赋值不会产生ArrayStoreException。现在我们有麻烦了。我们保存了List&lt;Integer&gt;实例到一个声明为仅仅存储List&lt;String&gt;实例的列表中。在第五行，我们从这个队列的单个列表中取得单个元素。编译器自动强转取到的元素到String，但是它是一个Integer，所以，我们在运行时获得ClassCastException。为了阻止这个发生，第一行(它创建了泛型队列)必须产生一个编译时错误。

像E, List&lt;E&gt;, and List&lt;String&gt;类型技术上被认为是不合具体化的类型[JLS, 4.7]。直观上来说，不可具体化类型是这样的类型，相对于编译时表示，它的运行时表示包含了更少的信息。因为擦除，可具体化的唯一参数化类型是像List<?>和Map<?,?>这样的非受限通配符类型(条目26)。创建非受限通配符类型的队列是合法的，虽然极少使用。

禁止泛型队列创建可能很恼人。这意味着，例如，泛型集合返回一个它的元素类型的队列，这通常是不可能的(但是为部分解决方案参考条目33)。这也意味着，当使用varargs方法(条目53)，与泛型类型结合，你将得到令人困惑的警告。如果这个队列的元素类型是不可具体化的，那么你将得到一个警告。SafeVarargs注解可以使用在解决这个问题(条目32)。

当你得到一个泛型队列创建错误或者一个对于队列类型强转的非受检强转警告，最好的解决方案是经常使用结合类型List&lt;E&gt;，而不是队列类型E[]。你可能牺牲简明或者性能，但是作为交换，你获得更好的类型安全和互操作性。

例如，假设你想要编写一个具有接受集合构造子和一个返回随机选择集合的元素的单个方法的Chooser类 。取决于你传入到构造子的何种集合，你可能使用chooser作为一个游戏骰子、魔法8球或者蒙地卡罗模拟器的数据来源。以下是一个没有泛型的简单实现：
```java
// Chooser - 一个极其需要泛型的类! 
public class Chooser { 
	private final Object[] choiceArray;

	public Chooser(Collection choices) { 
		choiceArray = choices.toArray(); 
	}

	public Object choose() { 
		Random rnd = ThreadLocalRandom.current(); 
		return choiceArray[rnd.nextInt(choiceArray.length)]; 
	}
}
```
为了使用这个类，每次使用这个方法的时候，你不得不把choose的返回类型从Object强转为需要要的类型，而且如果你得到类型错误，这个强转将会在运行时失败。把条目29的建议记到心里，我们尝试着修改Chooser使得它是泛型。改变如下粗体所示：
```java
// A first cut at making Chooser generic - won't compile 
public class Chooser<T> { 
	private final T[] choiceArray;
	public Chooser(Collection<T> choices) { 
		choiceArray = choices.toArray(); 
	}

	// choose method unchanged
}
```
如果你编译这个类，你会得到这个错误信息：
```java
Chooser.java:9: error: incompatible types: Object[] cannot be 
converted to T[]
	choiceArray = choices.toArray(); 
								  ^ 
	where T is a type-variable:
	 T extends Object declared in class Chooser
```
没多大关系，你可以说，可以把Object队列强转为一个T队列：
```java
choiceArray = (T[]) choices.toArray();
```
这可以摆脱这个错误，但是你反而会有一个警告：

```java
Chooser.java:9: warning: [unchecked] unchecked cast 
		choiceArray = (T[]) choices.toArray(); 
										    ^
	required: T[], found: Object[]
	where T is a type-variable:
  T extends Object declared in class Chooser
```
编译器告诉你：你不能保证运行时强转的安全性，因为这个程序不会知道T类型代表着什么，基础，元素类型信息在运行时会从泛型中擦除。这个程序可以运行吗？当然，但是编译器不能证明这个。你可以自己证明，把证明放到注释之中，而且用注释取消这个警告，但是你最好消除这个警告的根源(条目27)。

为了消除未受检的强转警告，使用列表而不是队列。这个一个Chooser类的版本，编译时没有错误或者警告：
```java
// 基于列表的Chooser - 安全类型
public class Chooser<T> { 
	private final List<T> choiceList;

	public Chooser(Collection<T> choices) { 
		choiceList = new ArrayList<>(choices); 
	}

	public T choose() { 
		Random rnd = ThreadLocalRandom.current(); 
		return choiceList.get(rnd.nextInt(choiceList.size())); 
	}
}
```
这个版本有点啰嗦，而且或许有点慢，但是为了内心能够平静，你不会再运行时不会得到ClassCastException，这是值得的。

总之，队列和泛型有非常不同的类型规则。队列是协变的和具体化的；泛型是不变的和擦除的。结果是，队列提供了运行时类型安全而不是编译时类型安全，对于泛型反之。通常，队列和泛型混合的不好。如果你发现自己混合了它们，而且在编译时错误或者警告，你的第一反应应该是用列表替换队列。
