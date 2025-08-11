---
layout: post
title: XmlContentLoader [C#]
date: 2008-09-25
tags:
  - csharp
series: xmlcontentloader
---
As a subject of my first technical post, I've chosen a small utility class built on top of LINQ. I am pleased to work with this handy and powerful .NET Framework 3.5 feature, which is LINQ. I'll be writing about many issues related to Linq (LinqToXml, new C#3.0 in action, and many others...)

A few weeks ago, my team got a task to build a very simple wizard application based on Windows Forms. The main goal of the wizard was to free end-users from manually editing configuration files for our applications. It was straightforward, a 4 or 5-step wizard basically giving the ability to set some connection strings, email server settings, and some WCF stuff. My part of the task was to prepare a component responsible for loading and saving selected XML elements or attributes from a configuration file. As you can imagine, the file was pretty big, but we needed to change only some parts of it.

I decided to build a more general-purpose component, which I called XmlContentLoader. First, I'll show you a sample scenario for using it, followed by some interesting parts of the realization.

Assume that we have an XML file looking like this:

```xml
<Root>
   <A>some string...</A>
   <AA>
       <AAA>yo</AAA>
   </AA>
   <B>true</B>
   <BB></BB>
   <C>
       <C1 c1Attribute="some string..." >1</C1>
   </C>
   <D dAttribute="2008.02.02">
   </D>
</Root>
```

Now, we want to load some parts of the file into memory, change them, and save them back to the file. We assume that we want to write as little code as possible.

So we create the class (or structure) responsible for holding data from the XML file (please note that the class can already exist in our application), containing properties corresponding to appropriate XML file items (elements or attributes). The XML item is chosen by defining an XPath query returning the XML element and optional attribute name (in case of choosing an XML attribute).

```csharp
internal class XmlContentLoaderTest
{
   [XmlItem(@"/Root/A[1]")]
   [XmlItem(@"/Root/C/C1", "c1Attribute")]
   public string MyString { get; set; }

   [XmlItem(@"/Root/B[1]")]
   public bool MyBool { get; set; }

   [XmlItem(@"/Root/D[1]", "dAttribute")]
   public DateTime MyDateTime { get; set; }

   [XmlItem(@"/Root/C[1]/C1[1]")]
   public Decimal MyDecimal { get; set; }
}
```

As you can see above, one property can be mapped to many XML items. Once we have the data structure, we can load, modify and save XML file content.

```csharp
string filePath = "...";
var a = XmlContentLoader.Load<XmlContentLoaderTest>(filePath);

a.MyBool = !a.MyBool;
a.MyString = a.MyString + ".";
a.MyDecimal = a.MyDecimal + 1;
a.MyDateTime = a.MyDateTime.AddDays(1);

XmlContentLoader.Save(filePath, a);
```

It's all we can do with XmlContentLoader. Now let's see some details of the implementation.

XmlContentLoader has only three public methods.

```csharp
public static T Load<T>(string filePath)
   where T : new()
{
   return Load(filePath, new T());
}

public static T Load<T>(string filePath, T @object)
{
   if (!File.Exists(filePath))           
       throw new FileNotFoundException(string.Format("File {0} does not exist", filePath));

   XDocument document = XDocument.Load(filePath);
   foreach (var p in GetPropertyToXmlItemsMappings(@object, document,
       new { Property = Type<PropertyInfo>(), XmlItems = Type<XmlItem[]>() }))
   {
       p.Property.SetValue(@object,
           Converters[p.Property.PropertyType].ConvertFromString(p.XmlItems.First().Value),
           null);
   }

   return @object;
}

public static void Save(string filePath, object @object)
{
   if (!File.Exists(filePath))           
       throw new FileNotFoundException(string.Format("File {0} does not exist", filePath));

   XDocument document = XDocument.Load(filePath);
   foreach (var p in GetPropertyToXmlItemsMappings(@object, document,
       new { Property = Type<PropertyInfo>(), XmlItems = Type<XmlItem[]>() }))
   {
       object propertyValue = p.Property.GetValue(@object, null);
       foreach (var c in p.XmlItems)
           c.Value = Converters[p.Property.PropertyType].ConvertToString(propertyValue);
   }

   document.Save(filePath);
}

private static Dictionary<Type, TypeConverter> Converters { get; set; }

private static T Type<T>()
{
   return default(T);
}
```


The implementation of both `Load` and `Save` methods is very similar. We load XML document content into memory using the `XElement` class (added in the new API in Net3.5), then we just iterate through the collection of items returned from the `GetPropertyToXmlItemsMappings` method. Each loop iteration sets a new property value or gets the value from a property and stores it in the XML document object structure. What's worth noting is that we use the `TypeConverter` class to provide conversion between `System.String` type and property type.

We don't know yet what the type of the collection item is, so let's see the implementation of `GetPropertyToXmlItemsMappings` method.

```csharp
private static T[] GetPropertyToXmlItemsMappings<T>(object @object, XNode document,
   T mappingTypeShape)
{
   var q =
       from p in @object.GetType().GetProperties()
       where Attribute.IsDefined(p, typeof(XmlItemAttribute)) &&
             StoreTypeConverter(p.PropertyType) != null // check if type has TypeConverter
       select new
       {
           Property = p,
           XmlItems =
           (
               from XmlItemAttribute a in Attribute.GetCustomAttributes(p,
                   typeof(XmlItemAttribute))
               select GetXmlItem(document, a)
           ).ToArray() // Force execution 'GetXmlItem()' method 
                       // (check if all specified xml items exist)
       };

   return q.Cast<T>().ToArray(); // Force execution 'GetXmlItem()' method 
                                 // (check if all specified xml items exist)
}

private static XmlItem GetXmlItem(XNode document, XmlItemAttribute a)
{
   return a.IsAttribute
              ? new XmlItem(ExtractAttribute(document, a.ElementPath, a.AttributeName))
              : new XmlItem(ExtractElement(document, a.ElementPath));
}

internal class XmlItem
{
   private XElement Element { get; set; }
   private XAttribute Attribute { get; set; }
   internal string Value
   {
       get
       {
           if (Element != null)
               return Element.Value;
           return Attribute.Value;
       }
       set
       {
           if (Element != null)
               Element.Value = value;
           else
               Attribute.Value = value;
       }
   }

   public XmlItem(XElement element)
   {
       Element = element;
   }
   public XmlItem(XAttribute attribute)
   {
       Attribute = attribute;
   }
}
```

ReSharper is so smart to tell me: "Parameter 'mappingTypeShape' is never used". So, what for did I add it to the method signature? Is it needed or not? Yes, it is. Because I wanted to use an anonymous type inside the method body instead of defining an additional type myself (what if the compiler can do that for me ? ;) ), but outside of the method, I needed to know the type of collection item. I had to specify the type somehow. But how? It's an anonymous type. I gave the exact shape of type as I did inside the method. The compiler is smart enough to use the same generated anonymous type. Please don't ask me why I did it this way ... :)

I think `XmlContentLoader` can be a very useful utility when you need to change only some parts of an XML file in a typed way, without the necessity of using XML serialization. Additionally, when you use the PropertyGrid control, you can build a nice application in just a few lines of code.