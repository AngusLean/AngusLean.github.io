---
title: Java的四种引用，强弱软虚，用到的场景
date: 2016/4/11
updated: 2018/5/23
tags:
categories:
   - Programming Language
   - Java
   - 并发
---

## 1. strong reference
强引用应该是日常编程中使用最多的一种一种引用类型:


     StringBuffer buffer = new StringBuffer()

上面的代码创建了一个StringReference类的实例然后保存它的强引用在变量buffer上。 不过很明显的，我们关注的是**它为什么称为 Strong Reference以及它和Garbagecollector是如何相互影响的** . 如果一个强引用通过 强引用链 是可达的，那么它肯定不满足垃圾回收的条件，这也正是我们的代码能够正常运行的基本条件（想象一下在一个类中拥有另外一个类的实例，结果它隐式的被回收了）
应用里使用不能扩展类的是一个比较常见的情况， 这个类可能会被标记为`final`，或者其他更为复杂的手段：工厂方法返回的不知道具体实现类的接口。 考虑
>  It's not uncommon for an application to use classes that it can't reasonably extend. The class might simply be markedfinal, or it could be something more complicated, such as an interface returned by a factory method backed by an unknown (and possibly even unknowable) number of concrete implementations. Suppose you have to use a class Widget and, for whatever reason, it isn't possible or practical to extendWidget to add new functionality.
What happens when you need to keep track of extra information about the object? In this case, suppose we find ourselves needing to keep track of each Widget's serial number, but theWidget class doesn't actually have a serial number property -- and because Widget isn't extensible, we can't add one. No problem at all, that's what HashMapsare for:
serialNumberMap.put(widget, widgetSerialNumber);
 
> This might look okay on the surface, but the strong reference towidget will almost certainly cause problems. We have to know (with 100% certainty) when a particularWidget's serial number is no longer needed, so we can remove its entry from the map. Otherwise we're going to have a memory leak (if we don't remove Widgets when we should) or we're going to inexplicably find ourselves missing serial numbers (if we remove Widgets that we're still using). If these problems sound familiar, they should: they are exactly the problems that users of non-garbage-collected languages face when trying to manage memory, and we're not supposed to have to worry about this in a more civilized language like Java.
Another common problem with strong references is caching, particular with very large structures like images. Suppose you have an application which has to work with user-supplied images, like the web site design tool I work on. Naturally you want to cache these images, because loading them from disk is very expensive and you want to avoid the possibility of having two copies of the (potentially gigantic) image in memory at once.
Because an image cache is supposed to prevent us from reloading images when we don't absolutely need to, you will quickly realize that the cache should always contain a reference to any image which is already in memory. With ordinary strong references, though, that reference itself will force the image to remain in memory, which requires you (just as above) to somehow determine when the image is no longer needed in memory and remove it from the cache, so that it becomes eligible for garbage collection. Once again you are forced to duplicate the behavior of the garbage collector and manually determine whether or not an object should be in memory.



## 2.  Weak references
> A weak reference, simply put, is a reference that isn't strong enough to force an object to remain in memory. Weak references allow you to leverage the garbage collector's ability to determine reachability for you, so you don't have to do it yourself. You create a weak reference like this:
WeakReference<Widget> weakWidget = new WeakReference<Widget>(widget);
 
> and then elsewhere in the code you can useweakWidget.get() to get the actual Widgetobject. Of course the weak reference isn't strong enough to prevent garbage collection, so you may find (if there are no strong references to the widget) that weakWidget.get()suddenly starts returning null.
To solve the "widget serial number" problem above, the easiest thing to do is use the built-in WeakHashMap class.WeakHashMap works exactly like HashMap, except that the keys (not the values!) are referred to using weak references. If a WeakHashMap key becomes garbage, its entry is removed automatically. This avoids the pitfalls I described and requires no changes other than the switch fromHashMap to a WeakHashMap. If you're following the standard convention of referring to your maps via theMap interface, no other code needs to even be aware of the change.
Reference queues
Once a WeakReference starts returningnull, the object it pointed to has become garbage and the WeakReference object is pretty much useless. This generally means that some sort of cleanup is required;WeakHashMap, for example, has to remove such defunct entries to avoid holding onto an ever-increasing number of deadWeakReferences.
The ReferenceQueue class makes it easy to keep track of dead references. If you pass a ReferenceQueueinto a weak reference's constructor, the reference object will be automatically inserted into the reference queue when the object to which it pointed becomes garbage. You can then, at some regular interval, process the ReferenceQueue and perform whatever cleanup is needed for dead references.








https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references
http://stackoverflow.com/questions/3329691/understanding-javas-reference-classes-softreference-weakreference-and-phanto
