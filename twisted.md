If a Deferred is garbage-collected with an unhandled error (i.e. it would call the next errback if there was one), then Twisted will write the error’s traceback to the log file. [unhandled-errors]

[unhandled-errors]: http://twistedmatrix.com/documents/current/core/howto/defer.html#unhandled-errors
