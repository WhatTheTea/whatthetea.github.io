---
layout:     post
title:      "Threading and LinqPad saves the day"
date:       2024-09-19 20:37:00 +0300
categories: dotnet
---

> be Threading hard may

Today at work I stumbled upon kind of trivial, but interesting bug. 

There is desktop app developed using WinUI + Community Toolkit. Developer used `ObservableCollection<T>` from `System.Collection.ObjectModel` to provide business objects collection with update notifications.

Look at the following code snippet, can you spot here the problem?

{%- highlight cs -%}
. . .
ObservableCollection<TReference> SomeCollection = [];
. . .

foreach (var item in SomeCollection) 
{
    if (item is not null) 
    {
        // Read data from item
        Console.WriteLine(item);
    }
}
. . .
{%- endhighlight -%}

If you like to read the docs (I hope these people exist), you can event cite me a paragraph from [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api)

<details>
<summary>Remarks from <a href="https://learn.microsoft.com/en-us/dotnet/api/system.collections.objectmodel.collection-1.system-collections-icollection-syncroot?view=net-8.0#remarks">ICollection docs</a></summary>
&emsp;Enumerating through a collection is intrinsically not a thread-safe procedure. To guarantee thread safety during enumeration, you can lock the collection during the entire enumeration. To allow the collection to be accessed by multiple threads for reading and writing, you must implement your own synchronization.
</details>
TLDR; These collections are not thread safe, but guess what - C# tries to be safe, so enumerator implemented here and used by `foreach` statement will throw. Thus, if you try to call `Enumerator.MoveNext` if collection was changed from other thread - expect an exception :)

The first thought to fix this is locking our collection, so `ICollection` provides us with SyncRoot object, which by the docs can be used like this:
{%- highlight cs -%}
ICollection ic = ...;
lock (ic.SyncRoot) 
{
   foreach (var item in ic)
   {
        // Use items from collection
   }
}
{%- endhighlight -%}

It's okay, but I don't like this. What if you use this data for IO or even HTTP requests? Well, collection will be locked the whole time code in the `lock` block is executing.

The good friend of mine, who I'm grateful for my skill boost since winter of 2024, told me simple way to evade this scenario - you can just copy items to array and use it for data access. Think about it, this operation is much faster than actual work we need to implement. It can be implemented this way:

{%- highlight cs -%}
ICollection ic = ...;

foreach (var item in ic.ToArray())
{
    // Use items
}
{%- endhighlight -%}  

Good, but as you see, it takes a bunch of words to explain this. Imagine working environment, or just a pleasant evening when coworker text you about PR:
> What is this `.ToArray()` meant for in your fix?

You can explain everything, but it's not *that* fun. What about this little program called **LinqPad**? It's like C# Interactive and all those web compilers married with pyplot or something xD
There is numerous possibilities with it, but this evening I've got the most vanilla experience from it. You can just type in some code sample, dump needed data into output and run it!
![Linqpad shows us exception](/assets/img/posts/2024-09-19-linqpad-and-threads/linqpad_error.png)

Here we can see that just using `collection` simply errors out. Let's add `ToArray()`:

![Linqpad shows us exception](/assets/img/posts/2024-09-19-linqpad-and-threads/linqpad_success.png)

Voila! We just got sort of snapshot of this collection to operate on) I hope this tool will come handy to me once again and I will let you notice

Thanks for reading, happy coding)