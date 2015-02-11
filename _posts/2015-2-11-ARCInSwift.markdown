---
layout: post
title:  "Swift中的ARC"
date:   2015-02-11 10:53:00
categories: swift
---

这本来是一篇evernote笔记，为了填充我的贫瘠的博客内容，今天就把他搬过来了。

Swift的内存管理是使用OC中熟悉的**ARC**（自动引用计数）。   

注意：引用计数仅仅用来管理Class(引用类型)，其他的值类型是通过拷贝传递，不属于管理范围。

关于内存引用计数的基础这里就不再讲到了，请查看[维基百科](http://en.wikipedia.org/wiki/Reference_counting) [官方解释](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AutomaticReferenceCounting.html#//apple_ref/doc/uid/TP40014097-CH20-ID49)  

##强引用循环

引用计数这种管理模式有一个缺点：**引用循环**。  
什么是引用循环：根据维基百科的解释  
<blockquote>
The naive algorithm described above can't handle reference cycles, an object which refers directly or indirectly to itself. 一个对象直接或者间接的引用到了了自己。
</blockquote>

到底引用循环有什么问题， 举例如下：  
定义类  
{% highlight swift %}  
class Person {
    var name: String?
    init(name: String) {
        self.name = name
        println("Person :\(name) has initialized")
    }
    var apartment: Apartment?
    deinit {
        println("Person:\(name) has dealloced")
    }
}
class Apartment {
    let number: Int
    init(number: Int) { self.number = number }
    var tenant: Person?
    deinit { println("Apartment #\(number) is being deinitialized")
    }
}  
@end
{% endhighlight %}  
建立起一个引用循环，然后试图释放对象。 
{% highlight swift %}  
        var xiami: Person? = Person(name: "xiami")
        var apartment: Apartment? = Apartment(number: 1)
        xiami!.apartment = apartment
        apartment!.tenant = xiami
        //释放局部变量
        xiami = nil
        apartment = nil
{% endhighlight %}  
终端输出：  
<blockquote>
Person :xiami has initialized  
</blockquote>
这时你会发现这两个对象的内存都**没有被释放**。

##解决引用循环
*	既然是引用计数没有清零导致的，那么我们可以一个一个把循环计数给干掉（这种方法不是正常人使用的方法，请直接忽略看下一条）。
*	使用weak和unowned  
<blockquote>
weak: 声明为weak的不能形成强引用关系，和OC的weak一样不会使引用计数加1，同时weak声明的对象必须是可选的变量，可选是因为weak引用的值可以为nil,变量是可以改变。在对象被销毁的时候，swift会自动把引用设置为nil.  
</blockquote>
<blockquote>
unowned: 和weak相似unowned同样不会使引用计数加1，和weak不同的是，unowned引用假定其对象始终存在，所以unowned不能是可选的对象，然而swift不会把unowned的对象在被销毁的时候设置为nil。所以你如果使用了被销毁的unowned对象会发生运行时错误。 
</blockquote>
所以**weak**就类似于OC中的**weak**，**unowned** 类似于OC中的 **unsafe_unretained**和**assign**。他们都是不会使引用计数增加的引用，这样就不会存在循环引用了。  
*示例：（引用自iBooks）*
{% highlight swift %}
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    deinit { println("\(name) is being deinitialized") }
}
 
class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { println("Card #\(number) is being deinitialized") }
}
{% endhighlight %}  
这里，由于一方的引用使用了**unowned**。所以没用形成循环引用。  

  
###防止unowned引起的运行时错误
从设计上避免引用到被释放的对象，如下：
{% highlight swift %}
class Country {
         let name: String
         let capitalCity: City!
         init(name: String, capitalName: String) {
             self.name = name
             self.capitalCity = City(name: capitalName, country: self)
         }
}
 
class City {
         let name: String
         unowned let country: Country
         init(name: String, country: Country) {
             self.name = name
             self.country = country
         }
}
{% endhighlight %}  
这里可以看出capital作为county的一部分，只要不独立存在，就不会引用到被释放的对象。  
上面例子还示范了怎么在**初始化**的时候把**self作为参数传递到其他方法**：**implicitly unwrapped optional。**利用其假定self已被初始化完成。

##使用闭包时避免循环引用
闭包中的循环引用：如果你把闭包付给某实例一个属性，而闭包中又capture了这个实例（使用了方法或者属性）。  
即使没有形成循环引用，闭包对实例的强引用会延长实例的寿命，这在某些时候也是不应该出现的。

解决方法：定义**capture list**  
capture list是定义闭包的一部分，可以使用capture list 规定闭包capture的规则，capture list 在定义闭包的时候写在**参数之前**（如果没有参数就写在in之后）。capture list的格式：每一组由方括号包起来，再由逗号分隔。如：[unowned self], [weak someObj],…...