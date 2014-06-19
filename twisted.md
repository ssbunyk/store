## Reasons for Unhandled error in Deferred
If a Deferred is garbage-collected with an unhandled error (i.e. it would call the next errback if there was one), then Twisted will write the error’s traceback to the log file. ([Deferred Reference](http://twistedmatrix.com/documents/current/core/howto/defer.html#unhandled-errors))

Here is small example of such behaviour:

```python
from twisted.internet import reactor, defer

d = defer.Deferred() # create deferred
d.addCallback(lambda res: 1/res) # add callback which divides by value passed
reactor.callLater(1, d.callback, 0) # start calling callbacks after a second
                                    # passing zero should raise a zero division error

reactor.callLater(2, reactor.stop) # stop reactor in two seconds
reactor.run()
```

Will give us:

```
Unhandled error in Deferred:
Unhandled Error
Traceback (most recent call last):
  File "/opt/zenoss/lib/python/twisted/internet/base.py", line 1171, in mainLoop
    self.runUntilCurrent()
  File "/opt/zenoss/lib/python/twisted/internet/base.py", line 793, in runUntilCurrent
    call.func(*call.args, **call.kw)
  File "/opt/zenoss/lib/python/twisted/internet/defer.py", line 361, in callback
    self._startRunCallbacks(result)
  File "/opt/zenoss/lib/python/twisted/internet/defer.py", line 455, in _startRunCallbacks
    self._runCallbacks()
--- <exception caught here> ---
  File "/opt/zenoss/lib/python/twisted/internet/defer.py", line 542, in _runCallbacks
    current.result = callback(current.result, *args, **kw)
  File "test.py", line 4, in <lambda>
    d.addCallback(lambda res: 1/res) # add callback which divides by value passed
exceptions.ZeroDivisionError: integer division or modulo by zero
```

To silence this, we should add errback to deferred, which will not raise exception again.

```python
from twisted.internet import reactor, defer
from twisted.python.failure import Failure

def print_res(res):
    if isinstance(res, Failure):
        print res.value
    else:
        print res

d = defer.Deferred() # create deferred
d.addCallback(lambda res: 1/res) # add callback which divides by value passed
d.addErrback(print_res)  # <------------ errback added
reactor.callLater(1, d.callback, 0) # start calling callbacks after a second
                                    # passing zero should raise a zero division error

reactor.callLater(2, reactor.stop) # stop reactor in two seconds
reactor.run()
```

This will give us 

```
integer division or modulo by zero
```

## Deferred chains 
If you need one Deferred to wait on another, all you need to do is return a Deferred from a method added to addCallbacks. 

To illustrate this, we will use example with Perspective Broker ([from this StackOverflow question](http://stackoverflow.com/q/20293490/816449). First, here is the code for the server:

```python
from twisted.spread import pb
from twisted.internet import reactor

class Server(pb.Root):
    def remote_add_numbers(self, a, b):
        print 'adding:', a, b
        return a + b

if __name__ == '__main__':
    reactor.listenTCP(1111, pb.PBServerFactory(Server()))
    reactor.run()
```

And this is code for client:

```python
from twisted.spread import pb
from twisted.internet import reactor
from twisted.python.failure import Failure

def print_res(res):
    if isinstance(res, Failure):
        print res.value
    else:
        print res

def add_numbers(obj):
    obj.callRemote("add_numbers", 2, 2)

if __name__ == '__main__':
    factory = pb.PBClientFactory()
    reactor.connectTCP("localhost", 1111, factory)
    d = factory.getRootObject()
    d.addCallback(add_numbers)
    d.addCallback(print_res)
    d.addErrback(print_res)
    d.addCallback(lambda _: reactor.stop())

    reactor.run()
```

It fails with

```
None
Unhandled error in Deferred:
Unhandled Error
Traceback (most recent call last):
Failure: twisted.spread.pb.PBConnectionLost: [Failure instance: Traceback (failure with no frames): <class 'twisted.internet.error.ConnectionLost'>: Connection to the other side was lost in a non-clean fashion: Connection lost.
]
``` 
And here is why... Because `.getRootObject()` has to wait until a network connection has been made and exchange some data, it may take a while, so it returns a `Deferred`. We attach to that deferred `add_numbers` as callback. If and when the connection succeeds and a reference to the remote root object is obtained, this callback is run. The first argument passed to the callback is a remote reference to the distant root object.

With that object we could do `obj.callRemote("add_numbers", 2, 2)`. This also returns a deferred. That deferred has no callbacks and errbacks, and is not stored in any variable. `add_numbers()` returns `None`, then control flow goes to `print_res`, which prints `None` and then to `reactor.stop()`. Reactor is stopping and then script finishes it's run, but in the same time deferred returned by `callRemote` is still waiting for reply. Garbage collector calls destructor on that deferred, and it is forced to close connection in non clear fashion.

To fix this we need to force callbacks that follows `add_numbers` to wait for callback returned by `callRemote` to finish. To do that we use deferred chain, and just return deferred from callback. Then next callback will be called only when this receives result, and provided with that result.
