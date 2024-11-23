[合集 \- .NET9 \& C\#13(2\)](https://github.com)[1\..NET9 \- 新功能体验（一）11\-21](https://github.com/hugogoos/p/18559710):[楚门加速器](https://chuanggeye.com)2\..NET9 \- 新功能体验（二）11\-22收起
书接上回，我们继续来聊聊.NET9和C\#13带来的新变化。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241122164136553-822656340.png)


# ***01***、新的泛型约束 allows ref struct


这是在 C\# 13 中，引入的一项新的泛型约束功能，允许对泛型类型参数应用 ref struct 约束。


可能这样说不够直观，简单来说就是Span、ReadOnlySpan类型，我们直接看下面的代码示例：


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241122164145326-952578803.png)


在没有新的约束allows ref struct之前，Span是不能当参数传入的，直接编译错误，但是有了新约束则就可以支持Span参数了。


因此C\# 13 中引入了 where T : allows ref struct 泛型约束后使得我们可以对泛型参数类型进行更加精细的控制。通过这个特性，泛型方法或类就可以接受 ref struct 类型，如 Span 、ReadOnlySpan等，因为这些类型是在栈上分配内存，能够提供更高效的内存管理和更快的执行速度，所以这个新特性特别适用于高性能、内存密集型的泛型方法和类，可以有效避免堆分配和垃圾回收的开销。


# ***02***、ref struct接口


在 C\# 13 之前，ref struct 是无法实现接口的。 从 C\# 13 开始，ref struct可实现接口，但必须遵循 ref 安全性规则。 例如，由于需要装箱转换，因此无法将 ref struct 类型转换为接口类型。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241122164152953-271903707.png)


如上图，ref struct类型可以实现IInterface接口，但是当用IInterface接口去接收RefStructInterface类型时则直接编译报错，无论直接接收还是强制转换都是不支持的。


# ***03***、在异步方法中使用ref struct


从C\# 13开始，ref struct可以在异步方法中使用，但是有一个限制：它们不能在与 await 表达式同一个代码块中交互。这是为了避免 ref struct在跨越异步操作时引发内存安全问题，因为 ref struct 类型的实例通常存储在栈上，并且不能在异步操作中跨越栈帧。


下面代码是在异步方法中使用ref struct示例：



```
ref int Process(ref int x)
{
    return ref x;
}
//在异步方法中使用ref
async Task RefInAsync()
{
    var value = 0;
    await Task.Delay(0);
    ref var local = ref Process(ref value);
}

```

# ***04***、在迭代器中使用ref struct


从 C\# 13 开始，允许在迭代器方法中使用 ref struct，前提是满足以下条件：不能在包含 yield return 的代码段中使用它们。这是因为yield return 语句会导致方法的执行暂停并在以后继续执行。如果在这期间使用了ref struct，可能会导致这些类型的生命周期管理出现问题（例如跨越栈帧的切换）。为了避免这种问题，C\# 13 规定，如果要在迭代器方法中使用 ref struct，则不能在 yield return 语句所在的代码段中操作它们。


下面是在迭代器中使用ref struct示例代码：



```
ref int Process(ref int x)
{
    return ref x;
}
//在迭代器中使用ref
IEnumerable<int> RefInIterator(int[] array)
{
    for (var i = 0; i < array.Length; i++)
    {
        ref var v = ref Process(ref array[i]);
        yield return v;
    }
}

```

# ***05***、部分属性、部分索引器


早在C\#2就引入了部分类，在C\#3引入了部分方法，到现在C\#13又新增了部分属性和部分索引器。


这一改进这意味着允许属性和索引器可以跨越多个部分进行声明和实现。这给自动生成代码或分离关注点带来了极大便利，也更加灵活地生成和管理属性代码，特别适用于与源代码生成器等工具结合使用的场景。


以下是 C\# 13 中属性支持partial的示例：



```
public partial class PartialExamples
{
    //部分属性
    public partial int Capacity { get; set; }
    //部分索引器
    public partial string this[int index] { get; set; }
    //部分方法
    public partial string? TryGetItemAt(int index);
}
public partial class PartialExamples
{
    private List<string> _items = ["one", "two", "three", "four", "five"];
    //部分属性
    public partial int Capacity
    {
        get => _items.Count;
        set
        {
            if ((value != _items.Count) && (value >= 0))
            {
                _items.Capacity = value;
            }
        }
    }
    //部分索引器
    public partial string this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }
    //部分方法
    public partial string? TryGetItemAt(int index)
    {
        if (index < _items.Count)
        {
            return _items[index];
        }
        return null;
    }
}

```

# ***06***、foreach 支持Index


相信很多人都遇到过想要在foreach的时候获取集合元素当前索引，一般两种选择，一种自己维护一个变量，一种直接改用for。


而.NET9开始总算改变了这一现状，可以在foreach时候同时获取到当前元素及其索引。


我们下面看看Index()方法给我们带来了多大便利，代码如下：



```
//.NET 9 之前
 public void Loop()
 {
     List<string> items = ["张三", "李四", "王五"];
     var idx = 0;
     foreach (var item in items)
     {
         idx++;
         Console.WriteLine($"第{idx}个人名字是：{item}");
     }
 }
 //.NET 9
 public void LoopNew()
 {
     List<string> items = ["张三", "李四", "王五"];
     //直接获取索引、元素
     foreach ((int Index, string Item) in items.Index())
     {
         Console.WriteLine($"第{Index + 1}个人名字是：{Item}");
     }
 }
 //.NET 9
 public void LoopNew2()
 {
     List<string> items = ["张三", "李四", "王五"];
     //先获取元组后，再获取索引、元素
     foreach (var item in items.Index())
     {
         Console.WriteLine($"第{item.Index + 1}个人名字是：{item.Item}");
     }
 }

```

***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Planner](https://github.com)


