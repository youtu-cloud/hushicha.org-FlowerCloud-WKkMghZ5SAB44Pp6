[合集 \- .NET9 \& C\#13(3\)](https://github.com)[1\..NET9 \- 新功能体验（一）11\-21](https://github.com/hugogoos/p/18559710)[2\..NET9 \- 新功能体验（二）11\-22](https://github.com/hugogoos/p/18563166):[veee加速器](https://liuyunzhuge.com)3\..NET9 \- 新功能体验（三）11\-24收起
书接上回，我们继续来聊聊.NET9和C\#13带来的新变化。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091915018-358365723.png)


# ***01***、Linq新方法 CountBy 和 AggregateBy


引入了新的方法 CountBy 和 AggregateBy后，可以在不经过GroupBy 分配中间分组的情况下快速完成复杂的聚合操作，同时方法命名也非常直观，可以大大提升工作效率。


我们先以CountBy为例，简单实现一个小功能，统计不同年龄有多少人，代码如下：



```
public class Student
{
    public string Name { get; set; }
    public int Age { get; set; }
}
public void CountByExample()
{
    var students = new List
    {
        new Student { Name = "小明", Age = 10 },
        new Student { Name = "小红", Age = 12 },
        new Student { Name = "小华", Age = 10 },
        new Student { Name = "小亮", Age = 11 }
    };
    //统计不同年龄有多少人，两个版本实现
    //.NET 9 之前
    var group = students.GroupBy(x => x.Age);
    foreach (var item in group)
    {
        Console.WriteLine($"年龄为：{item.Key}，有：{item.Count()} 人。");
    }
    //.NET 9
    foreach (var student in students.CountBy(c => c.Age))
    {
        Console.WriteLine($"年龄为：{student.Key}，有：{student.Value} 人。");
    }
}

```

通过代码可以发现，老版本中必须先调用GroupBy分组再调用Count统计才可完成，而现在只需要调用CountBy即可。


我们再以AggregateBy为例子，看看新老版本中如何计算每个班级中各自学生总年龄，代码如下：



```
public class Student
{
    public string Name { get; set; }
    public string Grade { get; set; }
    public int Age { get; set; }        
}
public void AggregateByExample()
{
    var students = new List
    {
        new Student { Name = "小明", Grade = "一班", Age = 10 },
        new Student { Name = "小红", Grade = "二班", Age = 12 },
        new Student { Name = "小华", Grade = "一班", Age = 10 },
        new Student { Name = "小亮", Grade = "二班", Age = 11 }
    };
    //统计每个班级各自学生总年龄，两个版本实现
    //.NET 9 之前
    var old = students
       .GroupBy(stu => stu.Grade)
       .ToDictionary(group => group.Key, group => group.Sum(stu => stu.Age))
       .AsEnumerable();
    foreach (var item in old)
    {
        Console.WriteLine($"班级：{item.Key}，总年龄：{item.Value} 。");
    }
    //.NET 9
    foreach (var group in students.AggregateBy(c => c.Grade, 0, (acc, stu) => acc + stu.Age))
    {
        Console.WriteLine($"班级：{group.Key}，总年龄：{group.Value} 。");
    }
}

```

# ***02***、序列化加强


在System.Text.Json中，.NET 9为序列化提供了新的选项和一个新的单例。


## 1、缩进选项


现在可以通过JsonSerializerOptions新属性IndentCharacter和IndentSize，自定义写入 JSON 的缩进字符和缩进大小。看看下面代码。



```
static void Main()
{
    var options = new JsonSerializerOptions
    {
        WriteIndented = true,
        IndentCharacter = '\t',
        IndentSize = 2,
        //处理中文乱码
        Encoder = JavaScriptEncoder.Create(UnicodeRanges.All)
    };
    var json = JsonSerializer.Serialize(
        new Student { Name = "小明", Grade = "一班", Age = 10 },
        options
    );
    Console.WriteLine(json);
    Console.ReadKey();
}

```

代码执行效果如下：


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091901186-37223224.png)


## 2、默认 Web 选项单例


在之前的版本中如果想要以小驼峰命名规则序列化对象可以配合JsonProperty特性实现。现在则可以直接通过JsonSerializerOptions.Web单例直接实现。



```
var json = JsonSerializer.Serialize(
    new Student { Name = "xiaoming", Grade = "yinianji", Age = 10 },
    JsonSerializerOptions.Web
);
Console.WriteLine(json);

```

代码执行效果如下：


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091849343-1868082119.png)


# ***03***、Task新方法Task.WhenEach


在.NET9之前，如果我们有一个任务列表，并希望每个任务完成后立刻处理它，那么我们只能通过不停的调用Task.WaitAny()方法来实现，现在.NET9引入了Task.WhenEach方法，以一种更优雅、更高效的方式处理这种情况。


因为Task.WhenEach 返回 IAsyncEnumerable\>，因此可以配合await foreach语句在任务完成时对其进行迭代，下面我们一起看看示例。



```
async Task WhenEachAsync()
{
    //生成100个随机时间完成的任务列表
    var tasks = Enumerable.Range(1, 100)
                   .Select(async i =>
                   {
                       await Task.Delay(new Random().Next(1000, 5000));
                       return $"任务 {i} 完成";
                   })
                   .ToList();
    //.NET 9 之前
    while (tasks.Count > 0)
    {
        var completedTask = await Task.WhenAny(tasks);
        tasks.Remove(completedTask);
        Console.WriteLine(await completedTask);
    }
    //.NET 9
    await foreach (var completedTask in Task.WhenEach(tasks))
    {
        Console.WriteLine(await completedTask);
    }
}

```

# ***04***、新的 TimeSpan.From\* 重载


在.NET9之前TimeSpan类提供了几种From\*方法，可以使用double类型来创建TimeSpan对象。但是，由于double是基于二进制的浮点格式，因此固有的不精确性可能会导致错误。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091838043-1122070725.png)


为了解决这个问题，.NET 9 添加了新的重载方法，可以使用整数创建TimeSpan对象。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091829431-2127594544.png)


如下面这段代码：



```
TimeSpan timeSpan1 = TimeSpan.FromSeconds(value: 101.832);
Console.WriteLine($"timeSpan1 = {timeSpan1}");
// timeSpan1 = 00:01:41.8319999
TimeSpan timeSpan2 = TimeSpan.FromSeconds(seconds: 101, milliseconds: 832);
Console.WriteLine($"timeSpan2 = {timeSpan2}");
// timeSpan2 = 00:01:41.8320000

```

# ***05***、新的内置Swagger


从.NET9开始使用Scalar代替内置的Swagger（Swashbuckle），一方面是因为Swashbuckle项目维护不够积极，另一个方面也是内部希望更专业于OpenAPI的发展。不管原因如何，我们都要根据自己的情况做好选择。


下面我们来一起体验一下Scalar。


首先创建一个Web Api项目，然后安装Scalar.AspNetCore包，修改Prag代码如：



```
public static void Main(string[] args)
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Services.AddControllers();
    builder.Services.AddOpenApi();
    var app = builder.Build();
    // scalar/v1
    app.MapScalarApiReference(); 
    app.MapOpenApi();
    app.UseAuthorization();
    app.MapControllers();
    app.Run();
}

```

然后我们添加一个简单的控制器，添加增删改查四个方法，代码如下：



```
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };
    private readonly ILogger _logger;
    public OrdersController(ILogger logger)
    {
        _logger = logger;
    }
    [HttpGet(Name = "")]
    public IEnumerable Get()
    {
        return Enumerable.Range(1, 5).Select(index => new Order
        {
            Date = DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Price = Random.Shared.Next(-20, 55),
            Name = Summaries[Random.Shared.Next(Summaries.Length)]
        })
        .ToArray();
    }
    [HttpPost(Name = "")]
    public bool Post(Order order)
    {
        return true;
    }
    [HttpPut(Name = "{id}")]
    public bool Put(string id, Order order)
    {
        return true;
    }
    [HttpDelete(Name = "{id}")]
    public bool Delete(string id)
    {
        return true;
    }
}

```

然后我们允许代码看看效果：


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091817715-427812483.png)


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091810339-385285621.png)


卖相还是相当惊艳的，左侧是接口列表，左下角可以切换黑白两种风格主题，右侧是接口详情，同时还配备了多种语言请求格式。


我们点击右下角Test Request测试一下获取接口。


![](https://img2024.cnblogs.com/blog/386841/202411/386841-20241124091801666-214744635.png)


可以在左边填写好参数，添加最上面的Send，会看到右下角请求结果。更详细复杂的应用大家可以自己摸索摸索。


***注***：测试方法代码以及示例源码都已经上传至代码库，有兴趣的可以看看。[https://gitee.com/hugogoos/Planner](https://github.com)


