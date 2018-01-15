**Finalizer是不可预测的，常常是危险的，而且通常来说是不必要的**。它的使用会引起不稳定的行为、糟糕的性能和可移植性问题。Finalizer有一些正当的用法，我们将在这个条目后面讲述，但是一般来说，你应该避免使用它。在Java 9中，finalizer被弃用了，但是它仍然在Java库里面使用着。Java 9中finalizer的替代是cleaner。**Cleaner危险性比finalizer小，但仍然是不可预测的、慢的和通常来说不必要的**。

C++程序员被警告，不要把Java中的finalizer或者cleaner看成C++*析构子(destructor)*的类比。C++中，析构子是标准的方式回收和对象相关的资源，是一个和构造子必要的对应物。Java中，当对象不可达时，程序员方面不需要做特别的工作，垃圾回收器会回收和对象相关的内存。C++析构子也用来回收其他的非内存资源。在Java中，try-with-resources或者try-finally代码块也是用在这个目的(条目 9)。

finalizer和cleaner的缺点是，不能保证它们被立即执行[JLS, 12.6]。在一个对象成为不可到达的时间，和它的finalizer或者cleaner执行的时间之间，可能存在任意长的时间。这意味着，你**永远不要在finalizer或者cleaner里面做时间紧要的任何事情**。比如，依赖一个finalizer或者cleaner关闭文件，这是一个严重的错误，因为打开文件描述符是一个有限的资源。如果因为系统延迟运行finalizer和cleaner，而很多文件被打开着，那么一个程序可能失败因为它不再能打开文件。

 finalizer和cleaner被执行的及时性主要是一个垃圾回收算法的功能，这对于不同的实现有千差万别。一个代码的行为，依赖于finalizer或者cleaner执行的及时性，同样可能不尽相同。一个程序完美运行在JVM上，而且你测试了它，然后在一个你最重要客户喜欢的JVM上糟糕地失败了，这是完全有可能的。

延迟终止化不只是一个理论上的问题。为一个类提供finalizer可以任意延迟对它的对象的回收。一个同事调试了一个长期运行的GUI应用，以难以理解的OutOfMemoryError方式死亡。分析指出，在它死亡的时候，这个应用在finalizer队列上有成千个图形对象，刚好等着被终止和回收，不幸的是，finalizer线程比另外的应用线程运行在更低的优先级，所以对象没有以它们成为符合终止化条件的速度被终止。语言规范没有保证哪个线程执行finalizer，所以没有跨平台的方式来阻止这样的问题，只有避免使用finalizer。cleaner比finalizer在这一点上好些，因为类的作者有对他们自己cleaner线程的控制。但是cleaner仍然运行在后台，在垃圾回收器的控制之下，所以没有保证立即清理。

不仅是规范没有提供finalizer或者cleaner将立即运行的保证，而且也没有提供它们到底会不会运行的保证。在某些不可到达的对象上，一个程序终止而没有运行它们，这是完全有可能的，甚至很有可能的。因此，你应该**从不依赖一个finalizer或者cleaner来更新持久化状态**。比如，依赖finalizer或者cleaner来释放一个在共享资源(比如数据库)上的持久化锁，这是一个把你整个分布式系统戛然而止的非常好的方式。

不要被System.gc和System.runFinalization方法所引诱。它们可能会增加finalizer或者cleaner的执行几率，但是它们不能保证。这两个方法是曾经是声称这个保证：System.runFinalizersOnExit和它的邪恶孪生化身Runtime.runFinalizersOnExit。这些方法有致命的缺陷，已经被废弃数十年了 [ThreadStop]。

finalizer的另外一个问题是，在终止过程中一个未捕获异常是被忽略的，这个对象的终止化也会停止[JLS, 12.6]。未捕获异常使得其他对象处于破坏的状态。如果另外一个线程尝试使用这样的破坏对象，会导致任意非确定性的行为。通常地，一个未捕获异常将终止线程，然后打印堆叠信息，但是如果发生在finalizer中则不会，它甚至不会打印一个警告。cleaner没有这样的问题，因为使用cleaner的库有对它线程的控制。

**使用finalizer和cleaner有严重的性能惩罚**。在我的机器上，创建简单的AutoCloseable对象，用try-with-resources关闭它，和垃圾回收器回收它，这段时间是大约2 ns。相反，使用finalizer的时间增加到550 ns。换句话说，创建和用finalizer销毁对象，大约慢50倍。这种要是因为finalizer抑制了有效率的垃圾回收。如果你使用它们来清理类的所有实例(在我的机器上每个实例大约500 ns)，cleaner在速度上和finalizer是可比的，但是就像下面讨论的，如果仅仅使用它们作为安全网络，cleaner会更快。这这些情况下，创建，清理和销毁一个对象在我的机器上花了66 ns，这意味着，如果你不使用它，你将为安全网络保障花费一个因素5(而不是50)。

**finalizer有严重的问题：它们使得你的类受finalizer攻击**。finalizer攻击背后的思想很简单：如果一个异常从构造子或者它的序列化等同物(即readObject和readResolve方法(12章))抛出，恶意的子类的finalizer可以运行在部分构造的对象上，这个对象本应该胎死腹中。finalizer可以在静态域中记录对对象的引用，防止它被垃圾回收。一旦这个恶意的对象被记录，调用这个对象的任意方法是易如反掌的事情，这个对象起初从来不允许存在。**从构造子抛出一个异常应该足够防止一个对象存在。在finalizer面前，则不是**。这样的攻击可能有可怕的后果。final类是免于finalizer攻击的，因为没有人可以编写一个final类的恶意子类。**为了保护非final类免于finalizer攻击，编写一个final的不做任何事情的finalizer方法**。

那么，对于一个类，它的对象封装了需要终止的资源，比如文件或者线程，不要编写finalizer或者cleaner，你应该怎么做？仅仅**只要让你的类实现AutoCloseable**，当每个实例不再需要的时候，要求它的客户端调用它的close方法，甚至在面对异常时，通常用try-with-resources来保证终止(条目11)。一个值得提起的细节是，实例必须跟踪它是否被关闭：close方法必须在一个域中记录这个对象已经不再有效，而且其他的方法必须检测这个域，如果它们在对象关闭之后被调用，抛出一个IllegalStateException。

那么，如果有的话，cleaner和finalizer对什么有用呢？或许它们有两个正当的用途。一个是作为安全保障，以防资源的拥有者忽略了调用它的close方法。虽然没有保证cleaner或者finalizer立即运行(或者究竟会不会)，如果客户端没有怎么做，晚清资源空比没有清空要好。如果你考虑编写这样一个安全保障的finalizer，需要深思熟虑这个防护是否值得这个代价。一些Java库类，比如FileInputStream、FileOutputStream、ThreadPoolExecutor和java.sql.Connection，有作为安全保障的finalizer。

第二种适合的用途与对象的本地对等体（native peer）有关。本地对等体是一个本地对象，普通对象通过本地方法(native method)委托给一个本地对象。

cleaner的第二个适合的用途与对象的*本地对等体(native peer)*有关。本地对等体是一个本地(非Java)对象，一个普通对象通过*本地方法(native method)*委托给一个这个本地对象。因为本地对等体不是一个普通的对象，垃圾回收器不知道它，所以当它的Java对等体被回收时不能够回收它。cleaner或者finalizer可能是这个任务的一个合适方案，假如性能是可以接受的，而且本地对等体没有保留关键资源。如果性能是不可接受的，或者本地对等体保留了必须立即回收的资源，这个类应该有个close方法，就像前面描述的一样。

cleaner有点难于使用。下面是一个简单的展现这种技巧的Room类。让我们假设房间在它们被回收之前必须被清理。Room类实现了AutoCloseable，它的自动清理安全保障使用了一个cleaner，这个事实仅仅是一个实现细节。不像finalizer，cleaner不会污染公开的API：

```java
// 一个使用cleaner作为安全保障的autocloseable类 
public class Room implements AutoCloseable { 
	private static final Cleaner cleaner = Cleaner.create();

    // 需要清理的资源。必须不引用Room！
	private static class State implements Runnable { 
		int numJunkPiles; // 这个房间里面的垃圾堆数量

		State(int numJunkPiles) { 
			this.numJunkPiles = numJunkPiles; 
		}

		// 被close方法或者cleaner调用 
		@Override public void run() {
			System.out.println("Cleaning room");
			numJunkPiles = 0; 
		}
	}

	// 这个房间的状态，与我们的cleanable共享
	private final State state;

	// 我们的Cleanable. 当它符合垃圾回收条件时，清理房间
	private final Cleaner.Cleanable cleanable;

	public Room(int numJunkPiles) { 
		state = new State(numJunkPiles); 
		cleanable = cleaner.register(this, state); 
	}

	@Override public void close() { 
		cleanable.clean(); 
	}
}
```
静态内嵌State类持有资源，cleaner需要这个资源清理清理房间。在这个例子中，它仅仅是numJunkPiles域，表示这个房间的混乱程度。更逼真地，它可是是一个final长整形，包含了一个对本地对等体的指针。State实现了Runnable，而且它的run方法最多只被调用一次，由Cleanable调用，当在Room构造子中利用我们的cleaner，注册我们的State实例时，我们获得这个调用。run方法的调用y由这两件事情之一触发：通常，它由一个调用Room的close方法触发，这个close方法调用Cleanable的clean方法。如果客户端到Room对象适合垃圾回收的时候不能够调用close方法，cleaner将调用(希望如此)State的run方法。

State实例没有引用它的Room实例，这是很重要的。如果是这样的话，它会创建一个循环，将阻止Room对象适合垃圾回收(和自动被清理)。所以，State必须是一个静态嵌套类，因为非静态嵌套类包含对它们的宿主类实例(enclosing instance)的引用(条目24)。相似地，使用一个lambda表达式是不明智的，因为它们可能很容易地获取宿主类对象(enclosing object)的引用。

就像我们前面说过的，Room的cleaner仅仅作为一个安全保障来使用的。如果在 try-with-resource代码块中，客户端包围所有Room实例化，自动清理没有必要。这个行为良好的客户端显示了这个行为：
```java
public class Adult { 
	public static void main(String[] args) { 
		try (Room myRoom = new Room(7)) { 
			System.out.println("Goodbye"); 
		} 
	} 
}
```
就像你可以预料的，运行Adult程序打印了Goodbye，接着“Cleaning room”。但是这个行为有误的代码怎么样呢，代码永远不会清理它的房间？
```java
public class Teenager { 
	public static void main(String[] args) { 
		new Room(99); 
		System.out.println("Peace out"); 
	} 
}
```
你可能预想它会打印出“Peace out”，接着“Cleaning room”，但是在我的机器上，它不会打印“Cleaning room”，程序只是退出了。这是我们前面说过的不可预测性。Cleaner文档说，“在System.exit的期间，cleaner的行为是依赖特定实现的，关于清理是否调用，没有保证”。虽然文档没有说明，对于正常的程序退出，同样是适用的。在我的机器中，添加System.gc()这行到Teenager的main方法，足够使得它在退出前打印“Cleaning room”，但是这没有保证在你的机器上可以看见同样的行为。

总之，不要用cleaner，或者在早于Java 9的发布中的finalizer，除了作为安全保障或者终止非关键的本地资源之外。即使那样，当心不确定性和性能后果。
