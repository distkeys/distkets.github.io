---
layout: post
title: "Design Pattern - Singleton"
date: 2014-03-25 22:49
categories:
- Design Pattern

tags:
- '2014'
---
{% include toc %}

### Problem Statement

We create a class, and we know the classes can be instantiated once or twice or a thousand times, but what if you only want **one object** of that class.


### Solution

Before we dive into the solution, let's understand what design pattern is.

Design Patterns are well-tested solutions to common problems and issues we run into in software development. Think of them as best practices, suggestions for how you might arrange your classes and objects to accomplish a result.


#### First approach

For **one object**, a solution might be an *Abstract Class*, but remember an Abstract Class isn't allowed to be instantiated at all.


#### Second approach

Another approach might be why don't we not create more than one object? <br>
You could do that, but it's not really enforcing anything, you can't guarantee that behavior.
<br>


#### Singleton

In **Singleton Design Pattern**, we don't want to have to write code to instantiate the Singleton, we want to assume that it always exists, there is always one of them, and we just want to be able to ask for it from any other point of the program and get it.<br>


> With the Singleton Design Pattern, we ensure that a class only has one instance, and we provide just one way to get to it.

<br>

**Step 1**<br>
Mark constructor as *Private*

What this means is nobody can instantiate the class from outside, nobody can say they want to create a new object. This doesn't solve the problem that we need there to be one of them. We can't create it from the outside.

{% highlight c linenos %}
public class MySingleton {

        // private constructor - no object can instantiate
        private MySingleton() { }

        ...
        ...
}
{% endhighlight %}

<br>

**Step 2**<br>
Create a static variable in this class that holds a placeholder to a singleton object

This means a variable that can hold a singleton object, and there is nothing in it right now.

{% highlight c linenos %}
public class MySingleton {
        // placeholder for singleton object
        private static MySingleton _singleObj = null;

        // private constructor - no object can instantiate
        private MySingleton() { }

        ...
}
{% endhighlight %}

<br>
**Step 3**<br>
Create a static method say *getInstance()*.

Static methods can only access static data members. In this case, we don't have to have an instance of this class yet. The first thing it asks is, do I exist? Is there anything in that _singleObj variable, if it's null then instantiate a new MySingleton object.


{% highlight c linenos %}
public class MySingleton {
        // placeholder for singleton object
        private static MySingleton _singleObj = null;

        // private constructor - no object can instantiate
        private MySingleton() { }

        public static MySingleton getInstance() {
            // Do I exists ?
            if (_singleObj == null) {
                    // Instantiate object
                    _singleObj = new MySingleton();
            }
            return _singleObj;
        }

        // Other member functions
        ...
}
{% endhighlight %}


We're allowed to instantiate the object in *getInstance()*  from inside this class because we marked that constructor as *private*, which means this class is allowed to call it, but nobody else is.

<br>

> We are actually using a technique here called Lazy Instantiation, which means that until someone asks for this object, it doesn't exist. However, when they ask for it, we'll look for it. If it doesn't exist, we create it, and we store a reference to it so that when the next person asks, it's already there.

<br>

Finally, to invoke Singleton.

{% highlight c linenos %}
MySingleton single = MySingleton.getInstance();

single.otherMemberFunctions();
{% endhighlight %}

<br><br>
