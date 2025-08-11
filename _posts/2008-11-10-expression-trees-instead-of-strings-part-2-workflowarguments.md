---
layout: post
title: Expression tree instead of strings part 1 - WorkflowArguments, Mapper [C#]
date: 2008-11-10
tags:
  - csharp
series: expression-trees
---
{%- include /_posts/series-toc.md series-name="expression-trees" -%}

Last time I was writing about a utility class called ReflectionHelper, this time I'll show you two real scenarios where this class or just an expression tree has been used.

For the last couple of months, I've been working on a project built on top of Workflow Foundation. Now we have C#3.0, many things can be done smarter, simpler, quicker, better, or ... more cool ;) Let's look at a simple workflow class:

```csharp
class MyWorkflow : SequentialWorkflowActivity
{
    public int MyInt { get; set; }
    public string MyString { get; set; }            

    public MyWorkflow()
    {
        CodeActivity codeActivity = new CodeActivity("codeActivity1");
        codeActivity.ExecuteCode += delegate
        {
            Console.WriteLine("MyInt : " + MyInt);                    
            Console.WriteLine("MyString : " + MyString);
            MyInt++;
            MyString = "yo";
        };
        Activities.Add(codeActivity);
    }
}
```

I know that not all of you had the opportunity to use WF so I'll try to give you some basics. The most elementary component in WF is an activity. We can say that activity is just a class with 'Excute' method responsible for doing some action. Workflow process is also an activity containg another activities which are executed one by one (that's why it derives from SequentialWorkflowActivity). Our workflow has only one activity CodeActivity which raises ExecuteCode event when it's executed. Additionally our process manipulates its state stored in properties MyInt and MyString. Let's see how to start this process and initialize its state:

```csharp
using (WorkflowRuntime workflowRuntime = new WorkflowRuntime())
{                
    var arguments = new Dictionary<string, object>()
        {
            {"MyString", "hello"},
            {"MyInt", Math.Max(5, 10)},
        };

    WorkflowInstance instance = workflowRuntime.CreateWorkflow(typeof(MyWorkflow), arguments);
    instance.Start();
}
```

The problem is that method CreateWorkflow takes a dictionary where the key is a property name and the value is a value of the property. But what if someone changes the name or the type of the property someday ? The code will compile fine but at runtime we will get an exception. Since we have C#3.0 we can very easily resolve these two problems.

```csharp
var arguments = new WorkflowArguments<MyWorkflow>()
{
    { w => w.MyString, "hello" },
    { w => w.MyInt, Math.Max(5,10) }
};

WorkflowInstance instance = workflowRuntime.CreateWorkflow(typeof(MyWorkflow), arguments);

public class WorkflowArguments<TWorkflow> : Dictionary<string,object>
{
    public WorkflowArguments<TWorkflow> Add<T>(Expression<Func<TWorkflow, T>> property,
        T propertyValue)
    {
        var propertyInfo = Blog.Post002.ReflectionHelper.GetProperty<TWorkflow,T>(property);
        Add(propertyInfo.Name, propertyValue);
        return this;
    }
}
```

The same technique we be used to process parameters after execution.

```csharp
static void workflowRuntime_WorkflowCompleted(object sender, WorkflowCompletedEventArgs e)
{
    Console.WriteLine("workflowRuntime_WorkflowCompleted");
    var parameters = new WorkflowOutputParameters<MyWorkflow>(e.OutputParameters);

    Console.WriteLine(parameters.GetParameter( w => w.MyInt));
    Console.WriteLine(parameters.GetParameter( w => w.MyString));            
} 

public class WorkflowOutputParameters<TWorkflow> 
{           
    private Dictionary<string, object> Parameters { get; set; }
    
    public WorkflowOutputParameters(Dictionary<string, object> parameters)
    {           
        Parameters = parameters;
    }

    public T GetParameter<T>(Expression<Func<TWorkflow, T>> property)
    {
        var propertyInfo = Blog.Post002.ReflectionHelper.GetProperty<TWorkflow, T>(property);
        return (T)Parameters[propertyInfo.Name];
    }
}
```

The second example is a Mapper class. It's quite a common scenario when we map an instance of class A into an instance of class B. For example, we very often load some kind of DAL entity from the database into memory, then we map it to some kind of business entity. In many cases, the shape of both types is very similar; they have the same properties/fields, so the mapping code is very simple. It just transfers values of properties or fields from one object to another. The Mapper class is a very simple class that gives us the ability to record the mapping of two types to each other. Next, we can execute the mapping process specifying two instances of those classes and the mapping direction. Let's assume that we have two types:

```csharp
class CustomerDal
{
    public int Id { get; set; }
    public string Name { get; set; }
    public double Age { get; set; }            
    public char Gender { get; set; }
    public string City { get; set; }
    public string Street { get; set; }
}

class CustomerBO
{
    public int Id { get; private set;  }    // only getter
    public string Name { get; set; }        // exactly the same name
    public double Age;                      // field
    public char Sex { get; set; }           // another name
    public string Address { get; set; }     // more complicated mapping
}
```

Now, we want to map an instance of the `CustomerDal` type to an instance of `CustomerBo`. First, we need to define how the properties of types should be transformed. Once we have this, we can start mapping.

```csharp
var m1 = new Mapper<CustomerDal, CustomerBO>    // A type, B type
  {
      {a => a.Id, b => b.Id, Direction.A2B},    // member of type A, member of type B, mapping direction
      {a => a.Name, b => b.Name},               // by default map in both directions
      {a => a.Age, b => b.Age},                  {a => a.Gender, b => b.Sex},
      { (a, b, d) => b.Address = a.City + " " + a.Street , Direction.A2B} 
        // code snippet executed during mapping
  };
      
var customerDal = new CustomerDal { Id = 1, Name = "Michael", 
    City = "Chicago", Street = "W Washington", Age = 25, Gender = 'M'};
    
var customerBO = new CustomerBO();
m1.Map(customerDal, customerBO);

customerBO.Name = customerBO.Name + "!";
customerBO.Sex = 'F';
customerBO.Age = customerBO.Age + 1;
         
m1.Map(customerBO, customerDal);
```

Again, instead of specifying mapped properties in code as string literals, we use expression trees. Expression trees are safer, cleaner, and much more powerful. I showed you just two examples of real-life applications of ExpressionTrees that go beyond Linq. You saw the potential. Now imagine what you can do in your project with it.

The source code for both examples can be found [here](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=3723).