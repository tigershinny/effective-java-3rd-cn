当你用泛型编程时，你将会看见许多编译器警告：非受检强转警告、非受检方法调用警告、非受检参数化vararg类型警告和非受检转换警告。你多熟悉泛型一些，你将获得更少警告，但是不要期待这些新代码干净利索地编译。

许多非受检警告容易消除。例如，假设你不慎编写如下声明：

```java
Set<Lark> exaltation = new HashSet();
```
编译器会轻轻地告诉你做错了什么：
```
Venery.java:4: warning: [unchecked] unchecked conversion 
		Set<Lark> exaltation = new HashSet(); 
							   ^ 
	required: Set<Lark> 
	found: HashSet
```
然后你可以进行指示的改正，使得警告消失。记住，你实际上不需要指定类型参数，仅仅用*菱形操作子(diamond operator)*(<>)(在Java7引入)表示。然后编译器会*推导(infer)*正确的实际类型参数(在这个情形中，Lark)：

```java
Set<Lark> exaltation = new HashSet<>();
```
一些警告会更加难于清除。这个章节充满了这样警告的例子。当你遇见需要更多思考的警告时，你需要坚持不懈！**尽你所能地消除每个非受检警告**。如果你消除了所有警告，那么你敢保证你的代码是类型安全，这是一个非常好的事情。这意味着，你不会在运行时遇见ClassCastException，而且它增加了你的信心：你的代码将像预期的一样运行。

**如果你不能消除警告，但是你可以证明：引起警告的代码是类型安全的，那时(也只有直到那时)用@SuppressWarnings("unchecked")注解抑制警告。**如果你抑制警告，而没有首先证明这个代码是类型安全的，那么你给自己关于安全的错误感觉，但是它仍旧可能在运行时抛出ClassCastException。然而，如果你忽略了你知道是安全的非受检警告(而不是抑制它)，那么你不会知道一个新警告什么时候突然出现，而这个警告表示一个真正问题。新警告将会消失在所有你没有使之静默的错误警报当中。

从单个本地变量声明到整个类，SuppressWarnings注解可以使用在任何声明。**永远在尽可能小范围内使用SuppressWarnings注解**。通常，这将会是变量声明或者一个很短的方法或者构造子。永远不要在整个类上使用SuppressWarnings。这样做可能隐藏了关键的警告。

如果你发现自己在一个方法或者构造子上使用SuppressWarnings注解，这个方法或者构造子长于一行，那么你可以把它移动到一个本地变量声明。你可能不得不声明一个新的本地变量，但是这是值得的。例如，考虑如下toArray方法，它来自于ArrayList：
```java
public <T> T[] toArray(T[] a) { 
	if (a.length < size) 
		return (T[]) Arrays.copyOf(elements, size, a.getClass()); 
	System.arraycopy(elements, 0, a, 0, size); 
	if (a.length > size) 
		a[size] = null; 
	return a; 
}
```
如果你编译ArrayList，那么这个方法产生这个警告：
```
ArrayList.java:305: warning: [unchecked] unchecked cast
		return (T[]) Arrays.copyOf(elements, size, a.getClass()); 
								   ^ 
	required: T[] 
	found: Object[]
```
把SuppressWarnings注解放到返回语句上，这是不合法的，因为它不是一个声明[JLS, 9.7]。你可能想把注解放到整个方法上，但是不要这么做。反而，声明一个本地变量来保留返回值，而且注解它的声明，就像如此：
```
// 为了减少@SuppressWarnings范围而添加本地变量
public <T> T[] toArray(T[] a) {
	if (a.length < size) { 
		// 整个强转是正确的，因为我们创建的这个队列，是和传入的这个是相同类型的，即T[]。
		@SuppressWarnings("unchecked") 
		T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass()); 
		return result; 
	} 
	System.arraycopy(elements, 0, a, 0, size); 
	if (a.length > size) 
		a[size] = null; 
	return a;
} 
```
最后的方法干净地编译，而且减少了抑制非受检警告的范围。

**每次你使用@SuppressWarnings("unchecked")注解，添加一个注释，表明为什么这么做是安全的**。这将会帮助其他人理解这个代码，而且更为重要的是，它将会减少这个的几率：某人将会改变代码，导致这个计算不安全。如果你发现编写这样注释是困难的，保持思考。你可能最终理解，非受检操作终究是不安全的。

总之，非受检警告是重要的。不要忽略它们。每个非受检警告代表着运行时ClassCastException的可能。尽最大的努力消除这些警告。如果你不能消除非受检警告，而且你可以证明调用它的代码是安全的，那么用@SuppressWarnings("unchecked")注解在尽可能窄的范围内抑制这个警告。为在注释中抑制警告这个决定，标明基本的阐述。
