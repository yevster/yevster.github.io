---
layout: default
title: From Java to .NET Core. Part 2. Types
---
# From Java to .NET Core. Part 2. Types

*This article was originally published on the [Red Hat Developer Blog](https://developers.redhat.com/blog/2017/06/15/from-java-to-net-core-part-2-types/#more-436152).*

In my [previous post in the series](https://developers.redhat.com/blog/2017/05/17/from-java-to-net-core-part-1/), I discussed some fairly surface-level differences between C#/.NET and Java. These can be important for Java developers transitioning to .NET Core, to create code that looks and feels “native” to the new ecosystem. In this post, we dig beneath the surface, to understand .NET’s type system. It is my belief that, with Java in the rear view mirror, the .NET type system is more effective and enjoyable to write on. But you be the judge.

## 1. Don’t be so primitive

Quick, what does this line of code do?
```
int num = 3;
```
The answer depends on the language in which it occurs. In Java, it creates an instance of an int – a primitive type. Java primitives do not extend the top-level object class, ```java.lang.Object```, and must be replicated in a corresponding object instance (or “boxed”) when passed or assigned to something demanding an object reference. So calling ```myList.add(num)``` (where ```myList``` is an instance of ```java.util.List```) would create a new instance of the class ```java.util.Integer``` that would replicate the value 3.

In .NET, the same line of code does something entirely different. It creates a new instance of the class ```System.Int32```. ```int``` is just a laconic C# alias for the ```System.Int32``` class. And ```System.Int32``` is a descendant of ```System.Object```, .NET’s top-level object class. Correspondingly, the line ```myList.Add(num)``` in C#, where myList is an instance of ```System.Collections.Generic.List<int>``` would not instantiate a different class to contain the value of ```num```. There’s no need for it because ```num``` is already an object.


>**Note:** get to know [all of C#’s “built-in types”](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/built-in-types-table) and their aliases. It is the established C# convention that the aliases be used instead of the full class names.  Be especially mindful of bool and string. Seeing String instead of string in C# code is one of the surest ways of sniffing out a Java developer.

## 2. Deeply-Held Values

There’s actually a bit more nuance to the .NET type system than I indicated above. There are no true “primitives” in .NET. All data types are subtypes of ```System.Object```. But all those subtypes can be divided into two groups: Value Types and References Types.

Value Types are all the types that are derived from of ```System.ValueType```. This includes all the types with built-in aliases that we discussed above except String, as well as all structs and enums. Value types have two important distinctions:

Value type instances are passed and assigned by value. In the previous section, we compared adding an instance of an int to a list in Java and C#. What I left out is that in Java, it is the reference to the newly constructed java.lang.Integer that gets stored in the list. In .NET, it is the actual value, not a reference, that gets stored in the list. But perhaps the simplest illustration of assignment by value I can offer is this unit test:
```
int i = 3;
int j = i;
Assert.AreNotSame(i, j);
Assert.AreEqual(i, j);
```
This test passes because the second line of the snippet above assigns the value of ```i``` to ```j```, but ```j``` does not become a reference to the same object as ```i```. Because these are not references. value types are assigned by value. 

Oh, and don’t worry about our trusty old friend ```==```. It works exactly the way you would expect. This is because (and I hope you’re sitting down), C# allows operator overloading. So for the built-in value types (and String!), this operator has been overloaded to compare values, not references.

Values type instances can live on the stack or the heap. When they are local variables, they live on the stack. When they are fields in another object or structure, they live wherever that object or structure lives. So when you write ```int num = 3```, the value ```3``` lives in the execution stack. But when you add num to a list, the value is copied into the heap where the list lives.
Reference types, on the other hand, always live on the heap and are always passed by reference, just like regular objects in Java.

## 3. Think Outside the (Auto)box

In Java, autoboxing occurs when a primitive value is assigned to an object. For example:

```
Integer num = 3;
```

In this case, a new object of type java.lang.Integer is created in the heap to replicate the value of the primitive literal ```3``` in object form. As we have already discussed, in .NET, there are no primitives. ```3``` is a value type literal that does not need to be boxed in assigning to the corresponding ```System.Int32``` class. But, boxing does need to occur in the following code:
```
Object num = 3;
```
In this case, we’re assigning an instance of a value type to a variable of a reference type. So under the covers, the compiler will produce code that will create a reference type “box” to contain the value ```3```. As an Object, num will behave like any other reference type: It will live in the heap, it will be passed by reference, and it will be garbage-collected. Auto-boxing is expensive, so avoid it whenever possible.

## 4. Generic Greatness

Given what we just discussed about the expensiveness of autoboxing, we can reasonably conclude that [Java-style type erasure for generics](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html) is not an option. If a linked list of value types was equivalent in the compiled code to a linked list of reference types, then every value type instance added to that list would have to be autoboxed. This, as we discussed above, is expensive.

Fortunately, .NET generics liberate us from this and other cataclysms associated with type erasure by retaining their type parameters at runtime. For this reason, ```List<string>``` and ```List<int>``` are, at runtime, two different classes. Go ahead, see for yourself:
```
Console.WriteLine(new List<string>().GetType().FullName);
Console.WriteLine(new List<int>().GetType().FullName);
```
And here’s the result.
```
System.Collections.Generic.List`1[[System.String, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]
System.Collections.Generic.List`1[[System.Int32, System.Private.CoreLib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]
```
There’s some extra information in the brackets (assembly information) that we’ll discuss in a future post. What’s important for now is that the parameterized type is retained and available at runtime.

This means that functions with type parameters can actually reference these parameters the way they cannot in Java. For example:
```
public static IList<T> CreateNObjects<T>(int number)
where T: new()
{
    var resultList = new List<T>(number);
    for (int i=0; i < number; ++i)
    {
        resultList.Add(new T());
    }
    return resultList;
}
```
The above code produces a list of whatever type is passed as a type parameter. In Java, this would require passing a separate parameter containing the class to be instantiated. In .NET, the generic parameter is available at runtime, so no separate class parameter is needed. Also note the where clause, which allows us to place specific constraints on the type parameter and catch unsatisfactory type parameter values at compile time.

# In Conclusion

That was a bit of a deep dive, so let’s come up for air. Most of the time when you develop .NET applications, you can get by without thinking too deeply into the mechanics of the type system. So here are the key points to remember:

* Use C#’s built-in aliases for the standard types (```int```, ```bool```, ```string```, etc).

* There’s no autoboxing when adding any value type instance to a generic collection or passing to a typed parameter.

* There is autoboxing when casting a value type to object (such as adding to a non-generic collection).

* In .NET, the type information for generics is available at runtime.

* The ```==``` operator in C# by default compares object references. For the built-in types, it’s been overloaded to compare by value, just as with primitives in Java. The biggest exception is strings: the ```==``` operator compares these by value, not by reference.