# xq-promise
An implementation of the promise and fork-join patterns for async processing in ``XQuery 3.1`` with [BaseX][1].

* [What is it?](#what-is-it)
* [Why?](#why)
* [Thanks!](#thanks)
* [Installation](#installation)
 + [Declaration](#declaration)
 + [Version 0.8.2-BETA](#version-082-beta)
 + [Dependencies](#dependencies)
* [The Basics of a Promise](#the-basics-of-a-promise)
 + [defer](#defer)
 + [Callbacks](#callbacks)
   - [then](#then)
    - [done](#done)
    - [always](#always)
    - [fail](#fail)
 + [Adding callbacks](#adding-callbacks)
   - [During Creation](#during-creation)
    - [After creation](#after-creation)
    - [Chaining Helper Functions](#chaining-helper-functions)
    - [Multiple Callbacks per event](#multiple-callbacks-per-event)
 + [when](#when)
* [The Power of Promises and Parallel Execution](#the-power-of-promises-and-parallel-execution)
 + [fork-join](#fork-join)
   - [How to interact with shared resources](#how-to-interact-with-shared-resources)
     * [Never attempt to write to a database within a fork](#never-attempt-to-write-to-a-database-within-a-fork)
      * [Do not open disc resources (databases, files) from multiple forks.](#do-not-open-disc-resources-databases-files-from-multiple-forks)
      * [Other words of caution!](#other-words-of-caution)
   - [Advanced Forking](#advanced-forking)
     * [Compute size](#compute-size)
     * [Max forks](#max-forks)
     * [Fork in Fork?](#fork-in-fork)
 + [is-promise](#is-promise)
* [Limitations](#limitations)
* [Unit Tests](#unit-tests)
* [Shout Out!](#shout-out)

## What is it?
This library implements the [promise][0] pattern as seen in many other languages and frameworks. 
Most notably those in the javascript community, such as jQuery and Q.js. 

The pattern resolves around the idea of deferred execution through what is often called a 
``deferred`` object. When an action is deferred, it returns a function, known as ``promise`` that 
when executed, at a later time, will perform and return the results of the work it deferred. 

Since the work is deferred and can be executed at an arbitrary time. There is the ability to attach further processing at a later date but prior to actual execution, via callback functions. This may sound confusing and hard to imagine, but I 'promise' the examples that follow will make it clearer. No pun intended.

## Why?
The main driver behind implementing the promise pattern was to realize ``parallel`` execution of XQuery code within a single query. If this sounds enticing, keep reading!

## Thanks!
I want to give a quick thanks to the [BaseX][1] team for their wonder implementation of ``BaseX`` and ``XQuery``.
It is because of their hard work, code samples, stellar architecture and code readability that this module was made possible! 

## Installation
 
 1. Copy the ``xq-promise-x.jar`` into your ``basex\lib`` directory 
 2. Install the ``xq-promise.xqm`` module found at: ``https://github.com/james-jw/xq-promise/src/main/xquery/xq-promise.xqm``

Or use [xqpm][5] to do it for you:
```
xqpm xq-promise
```

### Declaration
To use the module in your scripts simple import it like so:

```xquery
import module namespace promise = 'https://github.com/james-jw/xq-promise';
```

### Version 0.8.2-BETA
This module is currently in Beta and should be used with caution. Especially in scenarios involving the
writing of sensitive data. 

### Dependencies
This module is dependent on [BaseX][1].

## The Basics of a Promise 
In it's current iteration the library includes several methods with several overloads. The methods are as follows:

* defer
* when
* is-promise
* fork-join
* fork

### defer
```xquery
defer($work as function(*), 
      $arguments as item()*, 
      $callbacks as map(*,function(*))*) 
  as function(map(*,function(*)*))
```

The signature may look daunting but the pattern is simple. Use the <code>defer</code> method to defer a piece of work for later execution by passing in a function item and the future arguments. Lets see how this works with an example:

```xquery
import module namespace promise = 'https://github.com/james-jw/xq-promise';
let $greet := function($name) {
   'Hello ' || $name || '!'
}
let $promise := promise:defer($greet, 'world')
return
  $promise
```
In the above example, we defer the execution of the ``$greet`` method until we return <code>$promise</code>. Upon execution of the final line we should see ``hello world!``.

But wait! 

If you examine the output. The value returned is: <code>function (anonymous)#1</code>. This is not at all what we want.

This is where the power of the promise pattern starts to be realized. Formost, as mentioned prior, a promise is a ``function``. To retrieve it's value, it must be called:

```xquery
$promise()
```
The above modifcation will result in the expected answer: <code>Hello world!</code>

### Callbacks
In the above example we deferred a simple piece of work and then learned how to execute it at a later time by passing in the empty sequence. Now let me introduce the real power of the [promise][0] pattern with <code>callbacks</code>

A ``callback`` is a function which will be executed on the success or failure of some defered work. The available callback events to subscribe to are:

#### then
The then helper functions accepts a method to call on success as well as one to call on failure. Neither is required. The symantics of the method are as defined by the Promise A+ specification.

It acts as a pipeline function for transforming the response over successive callback executions. Unlike the next two events, but similar to ``fail``, this method can alter the pipeline result, and generally does.

#### done
Called on success. 

This method has no effect on the pipeline result and thus it's return value will be discared. Its main
purpose is for reacting to successful deferred execution as opposed to affecting its outcome like ``then`` does.

A common use case for ``done`` is logging.

#### always
Operates the same as ``done``, except it also is called on the promise's failure, not only success.

#### fail
Called if the action fails. The `fail` callback will be provided the error as a `map(*)` with some 'additional' information. 

```xquery
map {
  'code': 'error code',
  'description': 'error description',
  'value': 'error value',
  'module': 'file',
  'line': 'line number',
  'column': 'column number',
  'additional': map {
     'deferred': 'Function item which failed. Can be used to retry the request',
     'arguments': 'The arguments provided to the failed deferred.'
   }
}
```

A failure occurs if any deferred work or callback function throws an exception. The `fail` callback allows handling and potentially mitigating these errors during a fork-join process. Without a fail callback an exception will go uncaught and cause the entire query to stop. In essense, adding a fail callback to a deferred chain, is equivalent to the `catch` in a try/catch clause. 

Also similar to the `catch` clause, the `fail` callback has the option of returning a value as opposed to propegating or throwing an error itself. 

```xquery
promise:defer($work) 
  => promise:fail(function ($err) {
      if($err?code = 'XQPTY0005') then 'I fixed it!'
      else fn:error(xs:QName('local:error'), 'Unfixable error!')
  })
```

In the above example we see that if the `$err?code` returned matches `XQPTY0005`, error will be mitigated and the result of that chain of work will be the value `I fixed it!`. 

The simply logic is, if a `fail` callback chain returns a value, the failure will be handled with the value returned from the fail callback replaced in the result. 

If no suitable replacement value exists, but the error should simply be ignored. The `fail` callback should return the `empty-sequence`.

```xquery
promise:defer($work) 
  => promise:fail(function ($err) {
      if($err?code = 'XQPTY0005') then () (: Ignore the error and return nothing :)
      else fn:error(xs:QName('local:error'), 'Unfixable error!')
  })
```

In this example the error code `XQPTY0005` will result in the empty sequence `()`.

Ultimately, if the error code is not `XQPTY0005` and thus the failure cannot be mitigated. Throwing an exception within the callback using ``fn:error`` will cause the enitre fork and query to cease. 

#### Multiple fail callbacks
If multiple fail callbacks are added, multiple levels of error handling can be achieved. If the first callback is unable to process the error, it can itself throw an exception, which the second callback will be provided. This will continue until either a callback returns a value instead of erroring, or no further fail callbacks exist. In this ladder case, the query will cease.

Here is an example of two callbacks
```xquery
promise:defer($work) 
    => promise:fail(function ($err) {
      if($err?code = 'XQPTY0005') 
      then 'I fixed it!'
      else fn:error(xs:QName('local:error'), 'Unfixable error!')}) 
    => promise:fail(function ($err) {
      if($err?description = 'Unfixable error!') 
      then 'Never say never!'
      else fn:error(xs:QName('local:error'), 'Its Fatal!')
  })
```

### Adding callbacks
There are two ways to add callbacks: 
* During a promise's creation
* After a promise's creation

#### After creation
The above example attached the ``callback`` during the call with ``defer``; however there is another, even more powerful way. 
A ``promise`` can have callbacks attached after its been created!

For example:
```xquery
(: Same $worker, $req as above etc... :)
let $extractListItems := function ($res as map(*)) { $res?list?* } 
let $error := function ($result as item()*) {
     trace($result, 'Request failed!') => prof:void()
}
let $retrieve := p:defer($worker, ($req, $uri))
          => p:then(parse-json(?))
          => p:fail($error)
let $extract = p:then($retrieve, $extractListItems)
return
   $extract()
```

Note the calls to `then` and `fail` using the arrow operator. These calls add additional callbacks to the callback chain of the existing promise returning an agumented promise. Because a new expanded promise is returned, concise chaining can be accomplished!

Also note how the ``$extractListItems`` callback is appended to the ``$retrieve`` promise, resulting in a new promise ``$extract``, which when executed, will initiate the full chain of callbacks!

#### Chaining Helper Functions
Four methods, matching the callback event names, exist for attaching callbacks in a chain fashion using the arrow operator. For example:

```xquery
let $retrieve := p:defer($worker, ($req, $uri))
       => p:then(parse-json(?))
       => p:then($extractlistItems)
       => p:always(trace(?))
       => p:done(file:write-text(?, $path))
       => p:fail($error)
return
   $retrieve()
```

Imagine we want to make a request using the standard ``http:send-request`` method and then extract the body in a single streamlined callback pipeline.

Here is how this could be accomplished using the <code>promise</code> pattern and a ``then`` callback:
```xquery
let $req := <http:request method="GET" />
let $request := http:send-request($req, ?)
let $extract-body := function ($res) { $res[2] }
let $promise := promise:defer($request, 'http://www.google.com') 
       => promise:then($extract-body)
return
  $promise()
```
In the above example we attached a ``then`` callback. This callback function has the ability to transform the output of it's parent ``promise``. With this in the mind, it should be clear that the ``$extract-body``'s return value will be retuned at the call to ``$promise()``. 

In this example, since the ``$extract-body's`` input will be the result of its parent ``promise``. The result will be the response body of the http request.

### when
Another critical method in the [promise][0] pattern is the ``when`` function.
```xquery
when($promises as function(map(*,function(*)), 
     $callbacks as map(*,function(*))*) 
   as function(map(*,function(*)))
```

The purpose of  ``when`` is to combine 2 or more promised actions into a single promise. This is extremly powerful. Like the ``defer`` method disscussed earlier, the ``when`` method also returns a deferred ``promise``, which accepts callbacks just the same.

For example:
```xquery
let $write-and-return-users:= function ($name, $users) as item()* {(
      file:write($name, $users),
      $users
)}
let $extractDocName := promise:defer(doc(?), $doc-uri) 
  => promise:then($extract-name)
let $extractUsers := promise:defer(json-doc(?), $uri) 
  => promise:then($extract-list-items) 
let $users:= promise:when(($extractDocName, $extractUsers))
               => promise:then($write-and-return-users)
               => promise:fail(trace(?, 'Requesting users failed: '))
})
return
    $users() ! trace(.?username, 'Retrieved: ')
```

In this example, we perform two deferred actions and then merge their results in the ``$write-and-return-users`` callback. Since this item is attached to the ``when's`` promise on the ``then`` callback, its result will be seen on the call to ``$users()``.

We could continue to attach callbacks as needed until we are ready. There is no limit.

## The Power of Promises and Parallel Execution
Hopefully its clear now: how to defer work for later execution, what a promise is, and how to join multiple promises. It still may not be entirely clear what the benefit this pattern has in the context of XQuery; however that is about to change.

### fork-join

Let me introduce two last methods, and the whole reason I wrote this library.

```xquery
fork-join($promises as function(*)*) as item()*
```

It is simple yet powerful. It accepts a sequence of promises, or single arity functions and executes them in a fork join fashion, spawning threads as needed depending on the work load, followed by rejoining the work on the main thread. 

As seen earlier, ``promises`` can be used to build up a piece of work for later execution. With this ability, coupled with ``fork-join``. Parallelized ``XQuery`` processing becomes a reality. 

Lets see how we can use this capability by comparing a simple example involving making http requests. The example will use the ``promise`` pattern but not ``fork-join`` just yet. 

```xquery
import module namespace promise = 'https://github.com/james-jw/xq-promise';
let $work := http:send-request(<http:request method="GET" />, ?)
let $extract-doc := function ($res) { $res[2] }
let $extract-links := function ($res) { $res//*:a[@href => matches('^http')] }
let $promises :=
  for $uri in ((1 to 5) !  ('http://www.google.com', 'http://www.yahoo.com', 'http://www.amazon.com', 'http://cnn.com', 'http://www.msnbc.com'))
  let $defer := promise:defer($work, $uri)
       => promise:then($extract-doc)
       => promise:done(trace(?, 'Results found: ')})
       => promise:then($extract-links)
  return 
    $promises ! .()
```

In the above example, we use promises to queue up 25 requests and then execute them in order with:
```xquery
 $promises ! .()
```
If you run this example in BaseX GUI and watch the output window, you will see the requests come in as the query executes. 
This is due to the addition of the ``trace? 'Results Found: '`` callback.

Also notice, only one request is executed at a time. Each request must wait for the full response and processing of the previous. 
This is a current limitation of BaseX, since by design it runs each query in its own ``single`` thread. There are several workarounds such as splitting up the work via a
 master query, or using a string concatenated XQuery expression to spawn another process. Although effective, all these workarounds require extra effort 
and multiple components. Additionally they leave the language's domain and the context of the current query..

Luckily, with the introduction of this module ``xq-promise``. This is no longer the case! Lets change the previous example so it uses the newly introduced ``fork-join`` method to speed up the process, by splitting the requested work into multiple threads before returning the final joined value.

Luckily the previous example already used ``defer`` so the change is only one line. Replace:
```xquery
$promises ! .()
```
which manually executes each promise on the main thread, with:
```xquery
promise:fork-join($promises)
```

If you watch this execute in BaseX you will quickly see its executing much faster, with multiple requests being processed at once. 

On my machine, the first example without ``fork-join`` took on average 55 seconds. With ``fork-join`` this time dropped to 6 seconds!

That is a clear advantage! Playing around with ``compute size`` and ``max forks``, which I will introduce shortly, I have been able to get this even lower, to around 2 seconds!!

Hopefully its clear now what the use cases are for `fork-join` and how to use it!

#### How to interact with shared resources
With any async process comes the possibility of synchronization problems. Fortunately, XQuery due to its immutable nature is naturally suited to this type of work. Additionally from my limited look at BaseX, the code is very thread safe. Add to this, the introduction of the promise pattern and safe multi-threading appears to be real. 

There are a few things to note however when using ``fork-join``

##### Never attempt to write to a database within a fork
Luckily this does ``not`` restrict you from writing to databases, it just means: compute in forks, write after you have rejoined.
Fortunately you can be sure everything returned from the ``fork-join`` operation is returned on the main thread and thus is safe! 

For example:
```xquery
(: lots of computations chaining :)
let $result := promise:fork-join($promises)
return
  for $result in $results
  return db:add($db, $path, $result)
```

##### Do not open disc resources (databases, files) from multiple forks. 

Now this may seem like a major limitation, but its not. You can still interact and even open these resources in callbacks, 
and thus parallelized forks, however be cautious to try and open a resource only once and hopefully in a single piece of work.

```xquery
let $compute := function ($doc) {
   for sliding window $w in string-to-codepoints($doc)
   start at $spos when true()
   end at $epos when $epos - $spos = 25
   return $w
}
let $promises := db:list() ! promise:defer(db:open(?), ., map {
     'then': $compute
}
return
  promise:fork-join($promises)
```

Its important to note that all callbacks will be executed in the fork they originated from. So in this case, opening each database and computing the windows will occur in each fork.
If you attached an additional callback after ``$compute``, it too would execute in its origin fork, and not the master thread. Amazing!

In regards to database access, or any resources for that matter. Notice how I ensure to only open one database per fork. 
Although this is not a strict limitation it is a recommendation.

As an alternative, queue up the large resource prior to the ``fork-join`` and use it in the callbacks:
```xquery
let $largeResource := doc('...')
let $compute :=  function ($res) {
  $res?*[. = $largeResource//name]
}
let $promises := ...
return
  promise:fork-join($promises)
```

##### Other words of caution!
* Not everything should be parallelized.

For example, disc writes and other operations should be handled with care when using ``fork-join``

#### Advanced Forking
Certain scenarios can be optimized by changing the:
* compute size - Number of deferred jobs to process per thread
* max forks - Max number of forked threads to allow at once.

##### Compute size
Setting compute size is done during the call to ``fork-join`` by providing an additional ``xs:integer`` argument

For example:
```xquery
promise:fork-join($promises, 1)
```

The above query sets the compute size to 1. 
* The default ``compute size`` is 2. 

Depending on the level of effort in performing an individual task, this option can
be highly beneficial. For example, when computing millions of small computations, it may be worthwhile to increase the value significanly. For example to ``1000``.

On the contrary, when doing very computationally expensive tasks it may be best to leave this option alone, or even lower it to 1.

##### Max forks
The following query sets the ``compute size`` to 1 and the ``max forks`` to 20:
```xquery
promise:fork-join($promises, 1, 20)
```

For some operations, such as http requests, this can decrease script execution time.
* By default max forks is equal to the number of processor cores.

Here is the complete signature:
```xquery
promse:fork-join($promises as function(*,map(*)), 
                 $compute-size as xs:integer?, 
                 $max-forks as xs:integer?) 
             as item()*
```

##### Fork in Fork?
Why not?! You may wonder if you can fork in a forked callback. The answer is YES! Generally this would not be advised however in certain 
scenarios this is beneficial. Since all fork-joins share the same pool, inner forks merely ensure every thread is
used to it's maximum. However with anything, too many can be detrimental and is dependent on the type of work being performed.  

 Here is an example:

```xquery
let $request := http:send-request($req, ?)
let $request-all-links := function ($res) {
  let $promises := $res//a/@href ! promise:defer($request, .)
  return
    promise:fork-join($promises)
}
let $work := 
  for $uri in $uris
  return promise:defer($request, $uri, map { 'then': $request-all-links })
return
  promise:fork-join($work)
```

In this case, since the inner ``fork-join`` simply makes lots of external requests, this may actually improve execution time.

## Limitations
With any async process their are limitations. So far these are the only noticed limitations:
* Updating database nodes in a callback

## Unit Tests
Clone the repo and run ``basex -t`` within the repo's directory to run the unit tests.

## Shout Out!
If you like what you see here please star the repo and follow me on [github][7] or [linkedIn][6]

Happy forking!!

[0]: https://api.jquery.com/category/deferred-object/
[1]: http://www.basex.org
[2]: https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html
[3]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html
[4]: http://docs.basex.org/javadoc/org/basex/query/QueryModule.html
[5]: https://github.com/james-jw/xqpm
[6]: https://www.linkedin.com/pub/james-wright/61/25a/101
[7]: https://github.com/james-jw
[8]: http://docs.basex.org/wiki/XQuery_3.0#Function_Items
