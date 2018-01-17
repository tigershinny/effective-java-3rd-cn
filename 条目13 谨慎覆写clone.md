Cloneable接口目的是作为类的一个*混入接口(mixin interface)*(条目20)，宣称这些允许克隆。不幸的是，它未能作为这个目的。它的主要缺点在于，缺少了clone方法，而且Object的clone方法是受保护的。没有诉诸于*反射(reflection)*(条目65)，你不能仅仅因为它实现了Cloneable而调用对象的clone。甚至反射调用可能失败，因为没有保证对象有一个可获取的clone方法。尽管这个缺点和其他的缺点，这个技巧在合理的广泛使用中，所以理解它是值得的。这个条目告诉你怎么实现一个行为良好的clone方法，在合适的时候讨论，而且提出替代方法。

考虑到Cloneable没有包含任何方法，那么它做什么的呢？它决定了Object的受保护的clone实现的行为：如果一个类实现了Cloneable，Object的clone方法返回这个对象逐个域的拷贝；否则它抛出CloneNotSupportedException。这是接口的非常不典型的用法，而且不是应该效仿的一个。通常地，实现一个接口说明一个类能为它的客户端做什么。这个情形中，它改变了超类的受保护方法的行为。

尽管文档没有说，**在实践中，一个实现了Cloneable的类被期待提供一个正常功能的公开clone方法**。为了达到这个目的，这个类和它的所有超类必须遵从一个复杂的、非强制的和轻文档的协议。这个最终机制是脆落的、危险的和*超语言的(extralinguistic)*：它创建了对象而没有调用一个构造子。

clone方法的通用协定是不牢固的。就在这里，从Object规范拷贝的：

> 创建和返回这个对象的一个拷贝。“拷贝”的确切含义可能取决于这个对象的类。通常的意图是，对于任意对象x，表达式 
> x.clone() != x  
> 将会是真，而且表达式 
> x.clone().getClass() == x.getClass()  
> 将会是真，但是这些不是绝对的要求。虽然通常情况是这样 
> x.clone().equals(x) 
> 将会是真，但是这不是绝对的要求。
> 按照惯例，这个方法返回的对象应该通过调用super.clone获得。如果一个类和它的所有超类(除了Object)遵从这个惯例，它将会是这个情形
> x.clone().getClass() == x.getClass() 
> 按照惯例，返回的对象应该独立于克隆的对象。为了获得这个独立，在对象返回之前，应该有必要改变super.clone返回对象的一个或者多个域。

这个机制与构造子链是稍微相似的，除了这不是强制的：如果一个类的clone方法返回了一个实例，这个实例不是通过调用super.clone获取的，而是通过调用一个构造子，那么编译器不会报错，但是如果那个类的一个子类调用super.clone，最终的对象将有错误的类，这阻止了clone方法正常工作。如果一个覆写了clone方法的类是final的，这个惯例可以安全地忽略，因为没有子类可以担心。但是如果一个final类有一clone方法，它没有调动super.clone，那么没有理由为这个类实现Cloneable，因为它没有依赖于Object的clone实现的行为。

假设一个类的超类提供了一个行为良好的clone方法，你想在这个类中实现Cloneable。首先，调用super.clone。你取回的对象将会是原件的完整功能的副本。在你的类中声明的任何域，将有相等于原件那些域的值，如果每个域含有一个初始值，或者不可变对象的应用，那么返回的对象可能就是你想要的，在这个情形中没有必要进一步处理。比如，这是条目11中的PhoneNumber类的情形，但是注意，**不可变的类应该从不提供一个clone方法**，因为它仅仅会鼓励浪费拷贝。尽管如此，下面是PhoneNumber一个clone方法的样子：

```java
// clone方法，为没有可变状态的引用这个类
@Override public PhoneNumber clone() { 
	try { 
		return (PhoneNumber) super.clone(); 
	} catch (CloneNotSupportedException e) { 
		throw new AssertionError(); // Can't happen 
	} 
}
```
为了这个方法起作用，PhoneNumber类的声明将被修改为表明它实现了Cloneable。尽管Object的clone方法返回Object，但是这个clone方法返回PhoneNumber。这样做是合法和合理的，因为Java支持*协变返回类型(covariant return type)*。换句话说，覆写方法的返回类型可以是被覆写方法返回类型的一个子类型。这样不需要在客户端强转。在返回Object的super.clone的返回值之前，我们必须强转它，而且强转保证一定成功。

super.clone的调用包含在一个try-catch代码块中。这是因为Object声明它的clone方法抛出了CloneNotSupportedException，这是个受检查的异常。因为PhoneNumber实现了Cloneable，所以我们知道super.clone的调用将会成功。样板代码的必要性表明CloneNotSupportedException应该是没有受检查的(条目71)。

如果一个对象含有引用可变对象的域，那么上面显示的简单的克隆实现可能是灾难性的。比如，考虑条目7的Stack类：

```java
public class Stack {
	private Object[] elements; 
	private int size = 0; 
	private static final int DEFAULT_INITIAL_CAPACITY = 16;

	public Stack() { 
		this.elements = new Object[DEFAULT_INITIAL_CAPACITY]; 
	}

	public void push(Object e) { 
		ensureCapacity(); 
		elements[size++] = e; 
	}

	public Object pop() { 
		if (size == 0) 
			throw new EmptyStackException(); 
		Object result = elements[--size]; 
		elements[size] = null; // 消除过期引用 
		return result; 
	}

	// 保证至少有一个元素的空间 
	private void ensureCapacity() {
		if (elements.length == size)
			elements = Arrays.copyOf(elements, 2 * size + 1); 
	}
}
```
假设你想使得这个类是可克隆的。如果clone方法仅仅返回super.clone()，那么产生的Stack实例将会在它的size域有正确的值，但是它的elements域，将会引用与原来Stack实例一样的队列。修改原件将会在克隆中破坏了这个不变量，反之亦然。你将很快发现，你的程序产生了无意义的结果或者抛出一个NullPointerException。

这种情形应该永远不会发生，作为调用Stack类的唯一构造子的结果。**实际上，clone方法是起一个构造子的作用；你必须保证它不会影响原来的对象，而且保证它在克隆上正确地创建不变量**。为了Stack的clone方法正常工作，它必须拷贝Stack的内部组件。这么做最容易的方式是在elements队列上递归地调用clone方法：

```java
// Clone method for class with references to mutable state 
@Override public Stack clone() {
	try { 
		Stack result = (Stack) super.clone(); 
		result.elements = elements.clone(); 
		return result; 
	} catch (CloneNotSupportedException e) { 
		throw new AssertionError(); 
	}
}
```
注意，我们没必要将elements.clone的结果强转到Object[]。在一个队列上调用clone方法返回一个队列，它的运行时和编译时的类型，同被克隆的队列的那些类型是相同的。这是复制一个队列的优选习惯用法。事实上，队列是clone附加功能的唯一强制使用。

同时注意到，如果elements域是final的，前面的解决方法不起作用，因为clone被禁止赋新值到域。这是一个基本的问题：就像系列化，**Cloneable架构，与引用可变对象的final域的正常使用，是不相容的**，除了在这些情况下：可变对象可以在对象和它的克隆之间安全地共享。为了使得一个类可克隆，移除一些域的final修饰符可能是必要的。

仅仅递归地调用clone不总是足够的。比如，假设你为哈希表编写一个clone方法，它的内部有桶队列组成，每个桶引用键值对链表中的第一项。为了性能，类单独实现了它自己的轻量级链表，而不是内部使用java.util.LinkedList：

```java
public class HashTable implements Cloneable { 
	private Entry[] buckets = ...;
	private static class Entry { 
		final Object key; 
		Object value; 
		Entry next;

		Entry(Object key, Object value, Entry next) { 
			this.key = key; 
			this.value = value; 
			this.next = next; 
		}
	} 
	... // 其余省略
}
```
假设你仅仅递归克隆桶队列，就像我们已经为Stack做的：
```java
// 已破坏的clone方法 - 导致共享的可变状态！
@Override public HashTable clone() {
	try { 
		HashTable result = (HashTable) super.clone(); 
		result.buckets = buckets.clone(); 
		return result; 
	} catch (CloneNotSupportedException e) { 
		throw new AssertionError(); 
	}
}
```
尽管克隆有它自己桶队列，但是这个队列与原来的队列引用同一个链表，这可能在克隆和原件中容易造成不确定的行为。为了解决这个问题，你不得不拷贝由每个桶组成的链表。下面是一个通用的方法：
```java
// 复杂可变状态类的递归clone方法
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry  next;

        Entry(Object key, Object value, Entry next) {
            this.key   = key;
            this.value = value;
            this.next  = next;  
        }
        // 递归地拷贝由这个Entry开头的链表
        Entry deepCopy() {
            return new Entry(key, value,
                next == null ? null : next.deepCopy());
        }
    }

    @Override public HashTable clone() {
        try {
	        HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    ... //其余省略
}
```
私有类HashTable.Entry被增强为支持一个“深度拷贝”方法。HashTable的clone方法，分配了新的恰当大小的一个桶队列，遍历了原来的桶队列，深拷贝了每个非空桶。Entry的deepCopy方法，递归地调用自己来拷贝由这个项开头的整个链表。如果桶不是太长，这个技巧是较好的而且工作正常，但是这不是克隆链表的一个好方式，因为它为列表中每个元素消耗了一个栈帧。如果列表很长，这可能容易造成栈溢出。为了防止这个发生，你可以用迭代代替deepCopy里面的递归：
```java
// 迭代地拷贝由这个Entry开头的链表
Entry deepCopy() {
   Entry result = new Entry(key, value, next);
   for (Entry p = result; p.next != null; p = p.next)
      p.next = new Entry(p.next.key, p.next.value, p.next.next);
   return result;
}
```
克隆复杂可变对象的最后方案是调用super.clone，设置最终对象中的所有域为它们的初始状态，然后调用更高层的方法重新生成原来对象的状态。对于我们HashTable例子的情形，buckets域将会初始化为一个新的桶队列，而且将会为克隆哈希表中的每个键值映射，调用put(key, value)方法(没有展示出来)。这个方法通常是一个简单而非常优雅的clone方法，它不会像直接操作克隆内部结构的clone方法运行那么快。虽然这个方法简洁，但是与整个Cloneable结构是相违背的，因为它盲目地覆盖逐个域的对象拷贝，它形成了这个架构的基础。

就像构造子，clone方法在创建中决不应该在克隆上调用一个可覆写的方法(条目19)。如果clone调用一个被子类覆写的方法，在子类有机会克隆中修改它的状态之前，这个方法将会执行，这很可能导致在克隆和原件中的损坏。所以，前面段落讨论的put(key, value)方法，应该是final或者私有的。(如果它是私有的，那么它大概是非final公开方法的“helper方法”。）

Object的clone方法声明为抛出CloneNotSupportedException，但是覆写方法是不必要的。**公开的clone方法应该省略throws子句**，因为不会抛出受检查异常的方法更容易使用(条目71)。

当为继承而设计类的时候你有两个选择(条目19)，但是不管你选择哪个，类不应该实现Cloneable。你可能选择通过实现一个方法模拟Object的行为，这个方法是正常功能的受保护clone方法，而且clone方法是被声明为抛出CloneNotSupportedException。这给子类自由：是否实现Cloneable，就像它们直接扩展了Object。或者，你可以选择不实现一个可行的clone方法，而且通过提供如下的反生成clone实现阻止子类实现这个接口：

```java
//不支持Cloneable的可扩展类的clone方法 
@Override 
protected final Object clone() throws CloneNotSupportedException { 
	throw new CloneNotSupportedException(); 
}
```
另一个细节需要注意。如果你编写实现了Cloneable的线程安全类，记住它的clone方法必须正确地同步，就像其他的方法(条目87)。Objective的clone方法不是同步的，所以即使它的实现在其他方法是满足的，但是你可能必须编写返回super.clone()的一个同步clone方法。

简要概括，所有实现Cloneable的类应该用公开方法覆写克隆，这个公开方法返回类型是这个类本身。这个方法应该首先调用super.clone，然后修改需要修改的域。通常这意味着，拷贝任何由对象内部“深层结构”组成的可变对象，而且用它们拷贝的引用代替clone的这些对象的引用。虽然这些内部拷贝经常可以递归调用克隆，但是这不总是最好的方案。如果类包含了只有原始类型的域或者对不可变对象的引用，那么域很可能不需要修改。这个规律也有例外情况。比如，一个代表系列码或者其它唯一ID的域，如果它是原始类型或者不可变，那么它需要修改。

所有这些复杂性真的是有必要吗？其实很少。如果你扩展已经实现了Cloneable的类，那么你几乎没有选择，只有实现一个运行良好的clone方法。否则，你通常最好提供对象拷贝的替代方法。**一个更好的对象拷贝方案是，提供一个拷贝*构造子(copy constructor)*或者*拷贝工厂(copy factory)***。一个拷贝构造子仅仅是一个接受单个参数的构造子，这个参数的类型是包含构造子的类，比如，
```java
// 拷贝构造子 
public Yum(Yum yum) { ... };
```
拷贝工厂是一个静态工厂(条目1)，类似于拷贝构造子：
```java
// 拷贝工厂 
public static Yum newInstance(Yum yum) { ... };
```
拷贝构造子方法和它的静态工厂变体相对于Cloneable/clone有许多优点：他们不依赖于风险大的语言之外的对象创建机制；它们不要求非强制遵循轻文档规范；它们不会与final域的正常使用相冲突；它们不会抛出不必要的受检查异常；而且它们不要求强转。

此外，拷贝构造子或者工厂可以接受一个参数，它的类型是类实现的一个接口。比如，按照惯例，所有通用目的的数据集实现提供了一个构造子，它的参数是Collection或者Map类型。基于接口的拷贝构造子和工厂，更恰当地叫做转换构造子和转换工厂，让客户端选择拷贝的实现类型而不是强制客户端接受原件的实现类型。比如，假设你有一个HashSet，s，而且你想拷贝它为一个TreeSet。clone方法不能提供这个特性，但是对于使用转换构造子是容易的：new TreeSet<>(s)。

考虑到与Cloneable关联的所有问题，新接口不应该扩展它，而且新可扩展类不应该实现它。对于final类实现Cloneable，虽然有更少危害，这应该看作一个性能优化，为极少数适合的情形保留(条目67)，一般来说，由构造子或者工厂提供的拷贝功能是最好的。这个规则的一个显著例外是队列，它最好由clone法拷贝。
