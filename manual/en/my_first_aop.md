---
layout: default
title: BEAR.Sunday | My First Aspect
category: My First - Tutorial
--- 

# My First Aspect

## Add the current time to the greeting 

Add the current time to the [greeting resource](my_first_resource.html). The final outcome is like this.

```
Hello, BEAR. It is 1:53 now !
```

Placing the time after the message like this example is an easy way of implementing this.

{% highlight php startinline %}
    public function onGet($name = 'anonymous')
    {
        $time = date('g:i');
        return "{$this->message}, {$name}". " It is {$time} now";
    }
{% endhighlight %}

How about if you had to add the current time like this to another 10 resources? 
Carry out on another 10 resources an 'Add the time info to the end of some message' process.  
We need to do the same process over and over again so lets make it a method.

{% highlight php startinline %}
    public function onGet($name = 'anonymous')
    {
        return "{$this->message}, {$name}". timeMessage();
    }
{% endhighlight %}

This brings consolidation and increases re-usability.

But we could do the same using a trait.

{% highlight php startinline %}
    use TimeMessageTrait;

    public function onGet($name = 'anonymous')
    {
        return "{$this->message}, {$name}". $this->getTimeMessage();
    }
{% endhighlight %}

It is the same.

Even though this brings us some consolidation, the number of methods being used needs to change.

Say not the time, after the greeting we want to change it to add the weather.
Shall we change `timeMessage` to `weatherMessage`?

Or shall we add after the message `postMessage` to be more generic?
We are starting to exhaust ourselves. 
It is not really a good way to use methods like this, to transversely deal with the same types of processing.

## Making this an Aspect 

When you use this kind of method to carry out transverse processing a rather difficult side of coding appears.
When you run code either before and/or after a process like logging, caching, transactions etc,
have you ever experienced the same sort of code being left all over the place? 

In processing like `begin, [query], (commit | rollback)`, even though
`[query]` is the the only thing that changes we write often this the same way all about the shop.

So how about implementing crosscutting processing to consolidate code and keep the original core process.
We combine the original message and the added message dynamically.

In in this example, the re-usability of the 'Adding the time information' (crosscutting) process is called an 'Aspect'.
The binding of this aspect and the original process is what we refer to as 'Aspect Orientated Programming'

## Reflective method invocation interceptor 

In BEAR.Sunday uses the interceptor pattern to bind this crosscutting process to the original process.  

The crosscutting process is not used inside the original method, the original method is snatched (intercepted) by the 
crosscutting process, the crosscutting process then calls the original method.

In the previous example you are not calling the a process made up from various methods in 'Adding time' (=crosscutting process) from the 'greeting' method call.
The main process is called from the time adding crosscutting method, then calls the greeting.

Can you see that the Master/Slave (Dependency Relationship) has been reversed. 
Where the logging code was called by the DB access code, 
now the DB access code is called by the logging code.

This time the original code being run is called by a method using reflection, this is the (Method Invocation) object.
When using this object the parameters being used can be inspected and you can run the it just as fully intended. 
We use this object and intercept it.

## Create an Interceptor 

Let's take a look at the code.
Let's first take the original methods crosscutting process, this is an interceptor.

First of all a crosscutting process interceptor that does nothing.

*apps/Demo.Sandbox/src/Interceptor/TimeMessage.php*

{% highlight php startinline %}
<?php
/**
 * Time message
 */
namespace Demo\Sandbox\Interceptor;

use Ray\Aop\MethodInterceptor;
use Ray\Aop\MethodInvocation;

/**
 * +Time message add interceptor
 */
class TimeMessage implements MethodInterceptor
{
    /**
     * {@inheritdoc}
     */
    public function invoke(MethodInvocation $invocation)
    {
        $result = $invocation->proceed();
        return $result;
    }
}
{% endhighlight %}

Run the original method（`$invocation->proceed()`）, and return its response.

Using `$invocation->proceed()` when we run the original method the time message is added at the end of it.

{% highlight php startinline %}
    public function invoke(MethodInvocation $invocation)
    {
        $time = date('g:i');
        $result = $invocation->proceed() . ". It is {$time} now !";

        return $result;
    }
{% endhighlight %}

## Bind this interceptor to a specific method. 

In this way a we have created an interceptor which a from crosscutting process 
triggers the original method to run. Next we will assign (bind) the aspect to specific methods.

Using annotations is the general way, not using them here can also be done easily.

Add this to the `configure` method in `apps/Demo.Sandbox/src/Module/AppModule.php`.

*apps/Demo.Sandbox/src/Module/AppModule.php*

{% highlight php startinline %}
    // time message binding
    $this->bindInterceptor(
        $this->matcher->subclassesOf('Demo\Sandbox\Resource\App\First\Greeting\Aop'),
        $this->matcher->any(),
        [$this->requestInjection('Demo\Sandbox\Interceptor\TimeMessage')]
    );
{% endhighlight %}

In this way the `TimeMessage` interceptor is bound to any method in the `Demo\Sandbox\Resource\App\First\Greeting\Aop` class or sub-class.

### Let's run it

```
$ php apps/Demo.Sandbox/bootstrap/contexts/api.php get app://self/first/greeting/aop?name=BEAR

200 OK
content-type: ["application\/hal+json; charset=UTF-8"]
cache-control: ["no-cache"]
date: ["Tue, 01 Jul 2014 11:53:50 GMT"]
[BODY]
Hello, BEAR. It is 1:53 now !
```

We have now successfully bound our greeting resource to our aspect which states the time.
The greeting resource has no knowledge of the fact that it is being overridden.
The crosscutting process has called the original code and rides on top of it.

This has not built a dependency on the greeting resource, 
it is also different to a crosscutting process that uses a trait, it is a dynamic binding.
This resource can be bound to other aspects, and the aspect can be bound to other resources.

The weaving of this aspect can not only be applied to a resource.
It can be applied to any other standard object.
However the original format changes to a `weaving` format, hence compatibility may be lost.
