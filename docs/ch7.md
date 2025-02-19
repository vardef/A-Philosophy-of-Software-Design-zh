# 第 7 章 不同的层，不同的抽象

> Chapter 7 Different Layer, Different Abstraction

Software systems are composed in layers, where higher layers use the facilities provided by lower layers. In a well-designed system, each layer provides a different abstraction from the layers above and below it; if you follow a single operation as it moves up and down through layers by invoking methods, the abstractions change with each method call. For example:

> 软件系统由层组成，其中较高的层使用较低层提供的功能。在设计良好的系统中，每一层都提供与其上，下两层不同的抽象。如果您通过调用方法来跟踪一个在层中上下移动的操作，那么抽象会随着每次方法调用而改变。例如：

- In a file system, the uppermost layer implements a file abstraction. A file consists of a variable-length array of bytes, which can be updated by reading and writing variable-length byte ranges. The next lower layer in the file system implements a cache in memory of fixed-size disk blocks; callers can assume that frequently used blocks will stay in memory where they can be accessed quickly. The lowest layer consists of device drivers, which move blocks between secondary storage devices and memory.
- In a network transport protocol such as TCP, the abstraction provided by the topmost layer is a stream of bytes delivered reliably from one machine to another. This level is built on a lower level that transmits packets of bounded size between machines on a best-effort basis: most packets will be delivered successfully, but some packets may be lost or delivered out of order.

---

> - 在文件系统中，最上层实现文件抽象。文件由可变长度的字节数组组成，可以通过读写可变长度的字节范围来更新该字节。文件系统的下一层在固定大小的磁盘块的内存中实现了高速缓存。调用者可以假定经常使用的块将保留在内存中，以便可以快速访问它们。最低层由设备驱动程序组成，它们在辅助存储设备和内存之间移动块。
> - 在诸如 TCP 的网络传输协议中，最顶层提供的抽象是从一台机器可靠地传递到另一台机器的字节流。这个级别建立在一个较低的级别上，它在机器之间尽最大努力传输有限大小的数据包:大多数数据包会成功传递，但有些数据包可能会丢失或传递顺序错误。

If a system contains adjacent layers with similar abstractions, this is a red flag that suggests a problem with the class decomposition. This chapter discusses situations where this happens, the problems that result, and how to refactor to eliminate the problems.

> 如果系统包含具有相似抽象的相邻层，则这是一个红色标记，表明类分解存在问题。本章讨论了发生这种情况的情况，导致的问题以及如何重构以消除问题。（如果一个系统中相邻的分层，存在了相似的抽象概念，这就表明分类拆解可能存在问题）

## 7.1 Pass-through methods 直通方法

When adjacent layers have similar abstractions, the problem often manifests itself in the form of pass-through methods. A pass-through method is one that does little except invoke another method, whose signature is similar or identical to that of the calling method. For example, a student project implementing a GUI text editor contained a class consisting almost entirely of pass-through methods. Here is an extract from that class:

> 当相邻的层具有相似的抽象时，问题通常以直通方法的形式表现出来。直通方法是一种很少执行的方法，除了调用另一个方法（其签名与调用方法的签名相似或相同）之外。例如，一个实现 GUI 文本编辑器的学生项目包含一个几乎完全由直通方法组成的类。这是该类的摘录：

```java
public class TextDocument ... {
    private TextArea textArea;
    private TextDocumentListener listener;
    ...
    public Character getLastTypedCharacter() {
        return textArea.getLastTypedCharacter();
    }
    public int getCursorOffset() {
        return textArea.getCursorOffset();
    }
    public void insertString(String textToInsert, int offset) {
        textArea.insertString(textToInsert, offset);
    }
    public void willInsertString(String stringToInsert, int offset) {
        if (listener != null) {
            listener.willInsertString(this, stringToInsert, offset);
        }
    }
    ...
}
```

13 of the 15 public methods in that class were pass-through methods.

> 该类别中 15 个公共方法中的 13 个是直通方法。

img Red Flag: Pass-Through Method img

A pass-through method is one that does nothing except pass its arguments to another method, usually with the same API as the pass-through method. This typically indicates that there is not a clean division of responsibility between the classes.

> 直通方法除了将参数传递给另外一个与其有相同 API 的方法外，不执行任何操作。这通常表示各类之间没有明确的职责划分。

Pass-through methods make classes shallower: they increase the interface complexity of the class, which adds complexity, but they don’t increase the total functionality of the system. Of the four methods above, only the last one has any functionality, and even there it is trivial: the method checks the validity of one variable. Pass-through methods also create dependencies between classes: if the signature changes for the insertString method in TextArea, then the insertString method in TextDocument will have to change to match.

> 直通方法使类变浅：它们增加了类的接口复杂性，从而增加了复杂性，但是并没有增加系统的整体功能。在上述四个方法中，只有最后一个具有极少的功能，即使有也微乎其微：该方法检查一个变量的有效性。直通方法还会在类之间创建依赖关系：如果针对 TextArea 中的 insertString 方法更改了签名，则必须更改 TextDocument 中的 insertString 方法以进行匹配。

Pass-through methods indicate that there is confusion over the division of responsibility between classes. In the example above, the TextDocument class offers an insertString method, but the functionality for inserting text is implemented entirely in TextArea. This is usually a bad idea: the interface to a piece of functionality should be in the same class that implements the functionality. When you see pass-through methods from one class to another, consider the two classes and ask yourself “Exactly which features and abstractions is each of these classes responsible for?” You will probably notice that there is an overlap in responsibility between the classes.

> 直通方法表明类之间的责任划分存在混淆。在上面的示例中，TextDocument 类提供了 insertString 方法，但是用于插入文本的功能完全在 TextArea 中实现。这通常是一个坏主意：某个功能的接口应该在实现该功能的同一类中。当您看到从一个类到另一个类的直通方法时，请考虑这两个类，并问自己“这些类分别负责哪些功能和抽象？” 您可能会注意到，各类之间的职责重叠。

The solution is to refactor the classes so that each class has a distinct and coherent set of responsibilities. Figure 7.1 illustrates several ways to do this. One approach, shown in Figure 7.1(b), is to expose the lower level class directly to the callers of the higher level class, removing all responsibility for the feature from the higher level class. Another approach is to redistribute the functionality between the classes, as in Figure 7.1(c). Finally, if the classes can’t be disentangled, the best solution may be to merge them as in Figure 7.1(d).

> 解决方案是重构类，以使每个类都有各自不同且连贯的职责。图 7.1 说明了几种方法。一种方法，如图 7.1（b）所示，是将较低级别的类直接暴露给较高级别的类的调用者，而从较高级别的类中删除对该功能的所有责任。另一种方法是在类之间重新分配功能，如图 7.1（c）所示。最后，如果无法解开这些类，最好的解决方案可能是如图 7.1（d）所示合并它们。

![](./figures/00015.jpeg)

Figure 7.1: Pass-through methods. In (a), class C1 contains three pass-through methods, which do nothing but invoke methods with the same signature in C2 (each symbol represents a particular method signature). The pass-through methods can be eliminated by having C1’s callers invoke C2 directly as in (b), by redistributing functionality between C1 and C2 to avoid calls between the classes as in (c), or by combining the classes as in (d).

> 图 7.1：直通方法。在（a）中，类 C1 包含三个直通方法，这些方法只调用 C2 中具有相同签名的方法（每个符号代表一个特定的方法签名）。可以通过使 C1 的调用方像在（b）中那样直接调用 C2，通过在 C1 和 C2 之间重新分配功能以避免在（c）中的类之间进行调用，或者通过组合在（d）中的类来消除直通方法。 。

In the example above, there were three classes with intertwined responsibilities: TextDocument, TextArea, and TextDocumentListener. The student eliminated the pass-through methods by moving methods between classes and collapsing the three classes into just two, whose responsibilities were more clearly differentiated.

> 在上面的示例中，职责交织的三个类为：TextDocument，TextArea 和 TextDocumentListener。学生通过在类之间移动方法并将三个类缩减为两个类来消除直通方法，这两个类的职责更加明确。

## 7.2 When is interface duplication OK? 什么时候可以有重复的接口？

Having methods with the same signature is not always bad. The important thing is that each new method should contribute significant functionality. Pass-through methods are bad because they contribute no new functionality.

> 具有相同签名的方法并不总是不好的。重要的是，每种新方法都应贡献重要的功能。直通方法很糟糕，因为它们不提供任何新功能。

One example where it’s useful for a method to call another method with the same signature is a dispatcher. A dispatcher is a method that uses its arguments to select one of several other methods to invoke; then it passes most or all of its arguments to the chosen method. The signature for the dispatcher is often the same as the signature for the methods that it calls. Even so, the dispatcher provides useful functionality: it chooses which of several other methods should carry out each task.

> 一个方法调用另一个具有相同签名的方法很有用的例子是调度器。调度器是一种方法，它使用自己的参数从其他几种方法中选择一种来调用；然后，它将其大部分或全部参数传递给选定的方法。调度程序的签名通常与其调用的方法的签名相同。尽管如此，调度程序还是提供了有用的功能:它选择其他几种方法中的哪一种来执行每个任务。

For example, when a Web server receives an incoming HTTP request from a Web browser, it invokes a dispatcher that examines the URL in the incoming request and selects a specific method to handle the request. Some URLs might be handled by returning the contents of a file on disk; others might be handled by invoking a procedure in a language such as PHP or JavaScript. The dispatch process can be quite intricate, and is usually driven by a set of rules that are matched against the incoming URL.

> 例如，当 Web 服务器从 Web 浏览器接收到传入的 HTTP 请求时，它将调用一个调度器来检查传入请求中的 URL 并选择一种特定的方法来处理该请求。某些 URL 可以通过返回磁盘上文件的内容来处理；其他的则可能通过调用诸如 PHP 或 JavaScript 之类的语言的程序来处理。分发过程可能非常复杂，通常由与传入 URL 匹配的一组规则来驱动。

It is fine for several methods to have the same signature as long as each of them provides useful and distinct functionality. The methods invoked by a dispatcher have this property. Another example is interfaces with multiple implementations, such as disk drivers in an operating system. Each driver provides support for a different kind of disk, but they all have the same interface. When several methods provide different implementations of the same interface, it reduces cognitive load. Once you have worked with one of these methods, it’s easier to work with the others, since you don’t need to learn a new interface. Methods like this are usually in the same layer and they don’t invoke each other.

> 只要每种方法都提供有用且独特的功能，几种方法都具有相同的签名是可以接受的。调度程序调用的方法具有此属性。另一个示例是具有多种实现方式的接口，例如操作系统中的磁盘驱动程序。每个驱动程序都支持不同类型的磁盘，但是它们都有相同的接口。当几种方法提供同一接口的不同实现时，它将减少认知负担。使用其中一种方法后，与其他方法一起使用会更容易，因为您无需学习新的接口。像这样的方法通常位于同一层，并且它们不会相互调用。

## 7.3 Decorators 装饰器

The decorator design pattern (also known as a “wrapper”) is one that encourages API duplication across layers. A decorator object takes an existing object and extends its functionality; it provides an API similar or identical to the underlying object, and its methods invoke the methods of the underlying object. In the Java I/O example from Chapter 4, the BufferedInputStream class is a decorator: given an InputStream object, it provides the same API but introduces buffering. For example, when its read method is invoked to read a single character, it invokes read on the underlying InputStream to read a much larger block, and saves the extra characters to satisfy future read calls. Another example occurs in windowing systems: a Window class implements a simple form of window that is not scrollable, and a ScrollableWindow class decorates the Window class by adding horizontal and vertical scrollbars.

> 装饰器设计模式(也称为“包装器”)是一种鼓励跨层复制 API 的模式。装饰对象接受现有对象并扩展其功能;它提供一个与底层对象相似或相同的 API，它的方法调用底层对象的方法。在第 4 章的 Java I/O 示例中，BufferedInputStream 类是一个装饰器:给定一个 InputStream 对象，它提供了相同的 API，但是引入了缓冲。例如，当它的 read 方法被调用来读取单个字符时，它会调用底层 InputStream 上的 read 来读取更大的块，并保存额外的字符来满足未来的 read 调用。另一个例子出现在窗口系统中:Window 类实现了一个不能滚动的窗口的简单形式，而 ScrollableWindow 类通过添加水平和垂直滚动条来装饰窗口类。

The motivation for decorators is to separate special-purpose extensions of a class from a more generic core. However, decorator classes tend to be shallow: they introduce a large amount of boilerplate for a small amount of new functionality. Decorator classes often contain many pass-through methods. It’s easy to overuse the decorator pattern, creating a new class for every small new feature. This results in an explosion of shallow classes, such as the Java I/O example.

> 装饰器的动机是将类的专用扩展与更通用的核心分开。但是，装饰器类往往很浅：它们引入了大量的样板，以实现少量的新功能。装饰器类通常包含许多直通方法。过度使用装饰器模式很容易，为每个小的新功能创建一个新类。这导致诸如 Java I/O 示例之类的浅层类激增。

Before creating a decorator class, consider alternatives such as the following:

> 创建装饰器类之前，请考虑以下替代方法：

- Could you add the new functionality directly to the underlying class, rather than creating a decorator class? This makes sense if the new functionality is relatively general-purpose, or if it is logically related to the underlying class, or if most uses of the underlying class will also use the new functionality. For example, virtually everyone who creates a Java InputStream will also create a BufferedInputStream, and buffering is a natural part of I/O, so these classes should have been combined.
- If the new functionality is specialized for a particular use case, would it make sense to merge it with the use case, rather than creating a separate class?
- Could you merge the new functionality with an existing decorator, rather than creating a new decorator? This would result in a single deeper decorator class rather than multiple shallow ones.
- Finally, ask yourself whether the new functionality really needs to wrap the existing functionality: could you implement it as a stand-alone class that is independent of the base class? In the windowing example, the scrollbars could probably be implemented separately from the main window, without wrapping all of its existing functionality.

---

> - 您能否将新功能直接添加到基础类，而不是创建装饰器类？如果新功能是相对通用的，或者在逻辑上与基础类相关，或者如果基础类的大多数使用也将使用新功能，则这是有意义的。例如，几乎每个创建 Java InputStream 的人都会创建一个 BufferedInputStream，并且缓冲是 I/O 的自然组成部分，因此应该合并这些类。
> - 如果新功能专用于特定用例，将其与用例合并而不是创建单独的类是否有意义？
> - 您可以将新功能与现有的装饰器合并，而不是创建新的装饰器吗？这将产生一个更深的装饰器类，而不是多个浅的装饰器类。
> - 最后，问问自己新功能是否真的需要包装现有功能：是否可以将其实现为独立于基类的独立类？在窗口示例中，滚动条可能与主窗口分开实现，而无需包装其所有现有功能。

Sometimes decorators make sense, but there is usually a better alternative.

> 有时装饰者很有意义，但通常有更好的选择。

## 7.4 Interface versus implementation 接口与实现

Another application of the “different layer, different abstraction” rule is that the interface of a class should normally be different from its implementation: the representations used internally should be different from the abstractions that appear in the interface. If the two have similar abstractions, then the class probably isn’t very deep. For example, in the text editor project discussed in Chapter 6, most of the teams implemented the text module in terms of lines of text, with each line stored separately. Some of the teams also designed the APIs for the text class around lines, with methods such as getLine and putLine. However, this made the text class shallow and awkward to use. In the higher-level user interface code, it’s common to insert text in the middle of a line (e.g., when the user is typing) or to delete a range of text that spans lines. With a line-oriented API for the text class, callers were forced to split and join lines to implement the user-interface operations. This code was nontrivial and it was duplicated and scattered across the implementation of the user interface.

> “不同层，不同抽象”规则的另一个应用是，类的接口通常应与其实现不同：内部使用的表示形式应与接口中出现的抽象形式不同。如果两者具有相似的抽象，则该类可能不是很深。例如，在第 6 章讨论的文本编辑器项目中，大多数团队都以文本行的形式实现了文本模块，每行分别存储。一些团队还使用 getLine 和 putLine 之类的方法围绕行设计了文本类的 API。但是，这使文本类使用起来较浅且笨拙。在较高级别的用户界面代码中，通常在行中间插入文本（例如，当用户键入内容时）或删除跨行的文本范围。通过用于文本类的面向行的 API，调用者被迫拆分和合并行以实现用户界面操作。这段代码很简单，并且在用户界面的实现中被复制和散布。

The text classes were much easier to use when they provided a character-oriented interface, such as an insert method that inserts an arbitrary string of text (which may include newlines) at an arbitrary position in the text and a delete method that deletes the text between two arbitrary positions in the text. Internally, the text was still represented in terms of lines. A character-oriented interface encapsulates the complexity of line splitting and joining inside the text class, which makes the text class deeper and simplifies higher level code that uses the class. With this approach, the text API is quite different from the line-oriented storage mechanism; the difference represents valuable functionality provided by the class.

> 文本类提供面向字符的接口时，使用起来要容易得多，例如，insert 方法可在文本的任意位置插入任意文本字符串（可能包括换行符），而 delete 方法则删除文本在文本中的两个任意位置之间。在内部，文本仍以行表示。面向字符的接口封装了文本类内部的行拆分和连接的复杂性，这使文本类更深，并简化了使用该类的高级代码。通过这种方法，文本 API 与面向行的存储机制大不相同。差异表示该类提供的有价值的功能。

## 7.5 Pass-through variables 传递变量

Another form of API duplication across layers is a pass-through variable, which is a variable that is passed down through a long chain of methods. Figure 7.2(a) shows an example from a datacenter service. A command-line argument describes certificates to use for secure communication. This information is only needed by a low-level method m3, which calls a library method to open a socket, but it is passed down through all the methods on the path between main and m3. The cert variable appears in the signature of each of the intermediate methods.

> 跨层 API 重复的另一种形式是传递变量，该变量是通过一长串方法向下传递的变量。图 7.2（a）显示了数据中心服务的示例。命令行参数描述用于安全通信的证书。只有底层方法 m3 才需要此信息，该方法调用一个库方法来打开套接字，但是该信息会通过 main 和 m3 之间路径上的所有方法向下传递。cert 变量出现在每个中间方法的签名中。

Pass-through variables add complexity because they force all of the intermediate methods to be aware of their existence, even though the methods have no use for the variables. Furthermore, if a new variable comes into existence (for example, a system is initially built without support for certificates, but you later decide to add that support), you may have to modify a large number of interfaces and methods to pass the variable through all of the relevant paths.

> 传递变量增加了复杂性，因为它们强制所有中间方法知道它们的存在，即使这些方法对变量没有用处。此外，如果存在一个新变量（例如，最初构建的系统不支持证书，但是您后来决定添加该支持），则可能必须修改大量的接口和方法才能将变量传递给所有相关路径。

Eliminating pass-through variables can be challenging. One approach is to see if there is already an object shared between the topmost and bottommost methods. In the datacenter service example of Figure 7.2, perhaps there is an object containing other information about network communication, which is available to both main and m3. If so, main can store the certificate information in that object, so it needn’t be passed through all of the intervening methods on the path to m3 (see Figure 7.2(b)). However, if there is such an object, then it may itself be a pass-through variable (how else does m3 get access to it?).

> 消除传递变量可能具有挑战性。一种方法是查看最顶层和最底层方法之间是否已共享对象。在图 7.2 的数据中心服务示例中，也许存在一个对象，其中包含有关网络通信的其他信息，这对于 main 和 m3 都是可用的。如果是这样，main 可以将证书信息存储在该对象中，因此不必通过通往 m3 的路径上的所有干预方法来传递证书（请参见图 7.2（b））。但是，如果存在这样的对象，则它本身可能是传递变量（m3 还将如何访问它？）。

Another approach is to store the information in a global variable, as in Figure 7.2(c). This avoids the need to pass the information from method to method, but global variables almost always create other problems. For example, global variables make it impossible to create two independent instances of the same system in the same process, since accesses to the global variables will conflict. It may seem unlikely that you would need multiple instances in production, but they are often useful in testing.

> 另一种方法是将信息存储在全局变量中，如图 7.2（c）所示。这避免了将信息从一个方法传递到另一个方法的需要，但是全局变量几乎总是会产生其他问题。例如，全局变量使得不可能在同一过程中创建同一系统的两个独立实例，因为对全局变量的访问会发生冲突。在生产中似乎不太可能需要多个实例，但是它们通常在测试中很有用。

The solution I use most often is to introduce a context object as in Figure 7.2(d). A context stores all of the application’s global state (anything that would otherwise be a pass-through variable or global variable). Most applications have multiple variables in their global state, representing things such as configuration options, shared subsystems, and performance counters. There is one context object per instance of the system. The context allows multiple instances of the system to coexist in a single process, each with its own context.

> 我最常使用的解决方案是引入一个上下文对象，如图 7.2（d）所示。上下文存储应用程序的所有全局状态（否则将是传递变量或全局变量的任何状态）。大多数应用程序在其全局状态下具有多个变量，这些变量表示诸如配置选项，共享子系统和性能计数器之类的内容。每个系统实例只有一个上下文对象。上下文允许系统的多个实例在单个进程中共存，每个实例都有自己的上下文。

Unfortunately, the context will probably be needed in many places, so it can potentially become a pass-through variable. To reduce the number of methods that must be aware of it, a reference to the context can be saved in most of the system’s major objects. In the example of Figure 7.2(d), the class containing m3 stores a reference to the context as an instance variable in its objects. When a new object is created, the creating method retrieves the context reference from its object and passes it to the constructor for the new object. With this approach, the context is available everywhere, but it only appears as an explicit argument in constructors.

> 不幸的是，在许多地方可能都需要上下文，因此它有可能成为传递变量。为了减少必须意识到的方法数量，可以将上下文的引用保存在系统的大多数主要对象中。在图 7.2（d）的示例中，包含 m3 的类将对上下文的引用作为实例变量存储在其对象中。创建新对象时，创建方法将从其对象中检索上下文引用，并将其传递给新对象的构造函数。使用这种方法，上下文随处可见，但在构造函数中仅作为显式参数出现。

![](./figures/00016.gif)

Figure 7.2: Possible techniques for dealing with a pass-through variable. In (a), cert is passed through methods m1 and m2 even though they don’t use it. In (b), main and m3 have shared access to an object, so the variable can be stored there instead of passing it through m1 and m2. In (c), cert is stored as a global variable. In (d), cert is stored in a context object along with other system-wide information, such as a timeout value and performance counters; a reference to the context is stored in all objects whose methods need access to it.

> 图 7.2：处理传递变量的可能技术。在（a）中，证书通过方法 m1 和 m2 传递，即使它们不使用它也是如此。在（b）中，main 和 m3 具有对一个对象的共享访问权，因此可以将变量存储在此处，而不用将其传递给 m1 和 m2。在（c）中，cert 存储为全局变量。在（d）中，证书与其他系统范围的信息（例如超时值和性能计数器）一起存储在上下文对象中；对上下文的引用存储在其方法需要访问它的所有对象中。

The context object unifies the handling of all system-global information and eliminates the need for pass-through variables. If a new variable needs to be added, it can be added to the context object; no existing code is affected except for the constructor and destructor for the context. The context makes it easy to identify and manage the global state of the system, since it is all stored in one place. The context is also convenient for testing: test code can change the global configuration of the application by modifying fields in the context. It would be much more difficult to implement such changes if the system used pass-through variables.

> 上下文对象统一了所有系统全局信息的处理，并且不需要传递变量。如果需要添加新变量，则可以将其添加到上下文对象；除了上下文的构造函数和析构函数外，现有代码均不受影响。由于上下文全部存储在一个位置，因此上下文可以轻松识别和管理系统的全局状态。上下文也便于测试：测试代码可以通过修改上下文中的字段来更改应用程序的全局配置。如果系统使用传递变量，则实施此类更改将更加困难。

Contexts are far from an ideal solution. The variables stored in a context have most of the disadvantages of global variables; for example, it may not be obvious why a particular variable is present, or where it is used. Without discipline, a context can turn into a huge grab-bag of data that creates nonobvious dependencies throughout the system. Contexts may also create thread-safety issues; the best way to avoid problems is for variables in a context to be immutable. Unfortunately, I haven’t found a better solution than contexts.

> 上下文远非理想的解决方案。存储在上下文中的变量具有全局变量的大多数缺点。例如，为什么存在特定变量或在何处使用特定变量可能并不明显。没有纪律，上下文会变成巨大的数据抓包，从而在整个系统中创建不明显的依赖关系。上下文也可能产生线程安全问题；避免问题的最佳方法是使上下文中的变量不可变。不幸的是，我没有找到比上下文更好的解决方案。

## 7.6 Conclusion 结论

Each piece of design infrastructure added to a system, such as an interface, argument, function, class, or definition, adds complexity, since developers must learn about this element. In order for an element to provide a net gain against complexity, it must eliminate some complexity that would be present in the absence of the design element. Otherwise, you are better off implementing the system without that particular element. For example, a class can reduce complexity by encapsulating functionality so that users of the class needn’t be aware of it.

> 添加到系统中的每一个设计基础设施，如接口、参数、函数、类或定义，都会增加复杂性，因为开发人员必须了解这个元素。为了使一个元素提供对抗复杂性的净增益，它必须消除在没有设计元素的情况下出现的一些复杂性。否则，您最好在没有该特定元素的情况下实现该系统。例如，一个类可以通过封装功能来降低复杂性，这样该类的用户就不必知道它了。

The “different layer, different abstraction” rule is just an application of this idea: if different layers have the same abstraction, such as pass-through methods or decorators, then there’s a good chance that they haven’t provided enough benefit to compensate for the additional infrastructure they represent. Similarly, pass-through arguments require each of several methods to be aware of their existence (which adds to complexity) without contributing additional functionality.

> “不同的层，不同的抽象”规则只是此思想的一种应用：如果不同的层具有相同的抽象，例如直通方法或装饰器，则很有可能它们没有提供足够的利益来补偿它们代表的其他基础结构。类似地，传递参数要求几种方法中的每一种都知道它们的存在（这增加了复杂性），而又不提供其他功能。
