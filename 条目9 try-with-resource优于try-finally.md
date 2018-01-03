Java库包含很多资源，这些资源必须手动调用close方法关闭。这样的例子包括InputStream、OutputStream,和java.sql.Connection。关闭资源常常被客户端忽略，可以预见会有可怕的性能后果。许多这些资源用finalizer作为安全保障，但是finalizer不会很好的工作(条目8)。

历史上，一个资源应该正确地关闭，即使面对一个异常或者返回，try-finally语句是最好的保证方式：
```java
// try-finally - 不再是关闭资源最好的方式!
static String firstLineOfFile(String path) throws IOException { 
	BufferedReader br = new BufferedReader(new FileReader(path)); 
	try { 
		return br.readLine(); 
	} finally { 
		br.close(); 
	} 
}
```
这可能看上去不太糟，但是当你添加第二个资源时会变得更糟：
```java
// 当用于多于一个资源时，try-finally 是丑陋的! 
static void copy(String src, String dst) throws IOException { 		
	try { 
		InputStream in = new FileInputStream(src); 
		try { 
			OutputStream out = new FileOutputStream(dst); 
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
这可能难于置信，但是优秀的程序员甚至多数时间会把这个弄错。一开始，我在Java Puzzlers [Bloch05] 88页把这个弄错了，而且几年没人注意到。事实上在2007年，Java库的close方法的使用，三分之二是错误的。

用try-finally语句关闭资源的正确代码，就像前面两个代码例子，甚至有微妙的缺陷。在两个try代码块和finally代码块中的代码可能会抛出异常。比如，firstLineOfFile方法中，由于底层物体设备的失败，调用readLine可能抛出一个异常，而且close方法调用可能因为同样的原因失败。在这些情形下，第二个异常完全掩盖了第一个。在异常栈信息中没有第一个异常的记录，在实际系统中这个可能使得调试非常复杂，通常为了诊断这个问题，第一个异常是我们想要看见的。为了第一个异常而抑制第二个，虽然编写这样的代码是可能的，事实上，没有人这么干，因为这太啰嗦了。

当Java 7引进了try-with-resource语句[JLS, 14.20.3]，这些问题全解决了。为了用在这个结构，一个资源必须实现AutoCloseable接口，这个接口包含一个返回空的close方法。Java库和第三方库中的许多类和接口现在已经实现或者扩展了AutoCloseable。如果你编写一个必须关闭资源的类，你的类也应该实现AutoCloseable。

下面是我们第一个例子使用try-with-resource的样子：
```java
// try-with-resources - 关闭资源的最好方式！
static String firstLineOfFile(String path) throws IOException {
	try (BufferedReader br = new BufferedReader( 
			new FileReader(path))) {
		return br.readLine();
	} 
}
```
下面是我们第二个例子使用try-with-resource的样子：
```java
// 多个资源的try-with-resources - 简短而明了
static void copy(String src, String dst) throws IOException { 
	try (InputStream in = new FileInputStream(src);
			OutputStream out = new FileOutputStream(dst)) { 
		byte[] buf = new byte[BUFFER_SIZE]; 
		while ((n = in.read(buf)) >= 0) 
			out.write(buf, 0, n); 
		}
}
```
不仅try-with-resource版本比原来更加简短和可读，而且它们提供了好得多的可诊断性。考虑firstLineOfFile方法。如果异常由两个readLine调用和(不可见的)close方法抛出，为了前面的异常，后面的抑制了。事实上，为了保护你真实想看到的异常，多个异常可以被抑制。这些抑制的异常没有被丢弃，它们打印在在一个堆栈信息里面，用一个注释说明它们被抑制了。你也可以用getSuppressed方法以编程方式获取它们，在Java 7中这个方法添加到了Throwable。

你可以把catch子句放在try-with-resource语句，就像你可以放在try-finally语句一样。这让你可以处理异常，而没有用另外一层嵌套来污染你的代码。作为有点特别的例子，下面是firstLineOfFile方法的一个版本，它不抛出异常，但是如果不能够打开或者读取文件，它返回一个默认值：

```java
// 有catch子句的try-with-resource 
static String firstLineOfFile(String path, String defaultVal) { 
	try (BufferedReader br = new BufferedReader( 
			new FileReader(path))) { 
		return br.readLine(); 
	} catch (IOException e) { 
		return defaultVal; 
	} 
}
```

这堂课是很清晰的：当涉及必须关闭的资源的时候，总是优先使用try-with-resource，而不是try-finally。这样的代码更加简短和清晰，而且它产生的异常也更加有用。对于使用必须关闭的资源的代码，try-with-resource语句使得正确编写这些代码更容易，而用try-finally是在实践中不可能的。
