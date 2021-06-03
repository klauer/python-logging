## Python Logging Notes

Python logging is confusing.

### Levels

```
CRITICAL = 50
FATAL = CRITICAL
ERROR = 40
WARNING = 30
WARN = WARNING
INFO = 20
DEBUG = 10
NOTSET = 0
```

To pass a message through, a log message must have the following relationship:
``record.level >= logger.level``

So:
1. A logger set to ``DEBUG`` would see ``DEBUG, INFO, WARNING, ...``.
2. LOWER levels are relatively less important for the user to see.
3. HIGHER levels are relatively more important for the user to see.
4. Consider the levels like a "high water marker", indicating the nearby lake
   is nearing overflowing at WARNING (30) and a flood is imminent at CRITICAL
   (50).

### ``isEnabledFor``

``isEnabledFor`` performs this check. 

``isEnabledFor`` works like the following:

1. if ``logger.disabled``, return immediately
2. Otherwise,
    1. Check ``manager.disable >= level`` (usually not the case)
    2. Check ``level >= logger.getEffectiveLevel()``

3. ``logger.getEffectiveLevel()``
    1. Find the first logger ``[self, self.parent, self.parent.parent, ...]``
       that has a ``level`` set.
    2. Return the level found in (a) or fall back to ``NOTSET``.

 For performance reasons, the per-level ``isEnabledFor`` information is
 **cached** in a dictionary ``logger._cache[level]``.

This cache is locked using a **module-level** ``threading.RLock``.

### Level conversion

```python
>>> logging.getLevelName(10)
'DEBUG'
```

But the same function **reverses** it, too!

```python
>>> logging.getLevelName("DEBUG")
10
```

So using something like the following would make it consistent:


```python
from typing import Union

def validate_log_level(level: Union[str, int]) -> int:
    """
    Return a logging level integer for level comparison.

    Parameters
    ----------
    level : str or int
        The logging level string or integer value.

    Returns
    -------
    log_level : int
        The integral log level.

    Raises
    ------
    ValueError
        If the logging level is invalid.
    """
    if isinstance(level, int):
        return level
    if isinstance(level, str):
        levelno = logging.getLevelName(level)

    if not isinstance(levelno, int):
        raise ValueError("Invalid logging level (use e.g., DEBUG or 6)")

    return levelno
```

Consistent numerical values:
```
>>> validate_log_level(10)
10

>>> validate_log_level("DEBUG")
10

>>> logging.getLevelName(validate_log_level(10))
'DEBUG'
```

## Manager

Never hear of the ``logging.Manager``? Me either.

Normally, it appears there is just one ``Manager`` instance.

It contains the following information:

1. The root logger node (``Logger.root``)
2. A method to disable logging **globally** below a certain level.
3. If a no-handler-warning has been emitted (no handler found and no fallback)
4. ``loggerDict`` - a mapping of ``{logger_name: Logger}``
5. ``loggerClass`` the default ``Logger`` class to create on ``getLogger``.
6. ``logRecordFactory`` the factory used to create records

### Global disable logging

While the manager takes care of this, the public API is

``logging.disable(level)``

This also forces a cache clearing, so that loggers have to find their
effective levels again.

### getLogger

``logging.getLogger`` is a shortcut for ``Logger.manager.getLogger``!

... with the exception of the root logger, that is returned directly
from ``logging.root``.

Some things to note:
1. You can override the default Logger class by doing ``logging.``
2. Custom ``Logger`` classes **must** be subclasses of ``Logger``.
3. ``manager.setLoggerClass`` takes precedence over ``logging.setLoggerClass``
    1. They are two separate state variables!

### Placeholders?

Internal API for managing parent/child logger relationships. Quoting the
source code:

> Get a logger with the specified name (channel name), creating it
> if it doesn't yet exist. This name is a dot-separated hierarchical
> name, such as "a", "a.b", "a.b.c" or similar.
>
> If a PlaceHolder existed for the specified name [i.e. the logger
> didn't exist but a child of it did], replace it with the created
> logger and fix up the parent/child references which pointed to the
> placeholder to now point to the logger.


## Records

### extras

``LogRecord`` ignores unknown kwargs, so where do ``extra``s get added as
attributes?

Extras get patched in by way of ``record.__dict__[key] = value``.


## Adapters

``LoggerAdapter`` is a simple way to add contextual information onto a logger.
The adapter itself, however, is not 100% API-compatible with the ``Logger``
class.

It should be used when you have multiple contexts (objects, etc.) utilizing the
same logger instance, but you want the generated logger records to be easily
differentiable.

## Filters

Only the following classes implement the ``Filterer`` interface:

* Handler
* Logger

### Interface of a filter

Filters should either be callable or have a callable attribute ``.filter()``.
Either should take **one** positional argument: the ``LogRecord``.

```
class MyFilter:
    def filter(self, record):
        return True

def filter(record):
    return True
```

``MyFilter()`` and ``filter()`` are both acceptable (and equivalent in this
simple example).


### Application of filters

Filters are applied uniformly for either handlers or loggers:

```python
should_handle = all(filter() for filter in logger.filters)
should_handler_0_emit_record = all(filter() for filter in logger.handlers[0].filters)
```

### Handler filter

A **filter applied to a handler** takes in a record.
If **any** filter returns ``False``, the final message will not be emitted.

## Log Messages

### Hierarchy

The parent logger of ``logging.getLogger("a.b")`` is ``logging.getLogger("a")``.

### Propagation

Basic rule is that if:

```python
logger.propagate = False
```

That the log message will not propagate to `logger.parent`.
There's more to it than this, though.

### Effective Level

Unset log levels are configured as ``logging.NOTSET = 0``

### Flow [Logger version]

1. ``logger.debug("Message")``
    1. Ensure ``logger.isEnabledFor(DEBUG)``
2. ``logger._log()`` 
    1. Create a ``LogRecord`` by way of ``logger.makeRecord()``
    2. This is where exception information and stack information is determined.
3. ``logger.handle(record)``
    1. Stop if ``logger.disabled``
    2. Stop if ``not logger.filter(record)``
4. ``logger.callHandlers(record)``
    1. For each handler in ``logger.handlers``
        1. If the **handler** level is ``<=`` the ``record.levelno``, pass the
            message to the handler: ``handler.handle(record)``
    2. Repeat (a) for each parent/ancestor logger **if** ``.propagate``
    3. If **zero** handlers were found in this step, use the "last resort" logger.
       The log level number requirement still applies for the last resort logger.

For each handler that passed step (4)(a)(1):

1. Stop if ``not handler.filter(record)`` 
    1. All filters must return ``True``
2. Emit the record by way of:
    1. Acquire ``threading.RLock`` dedicated to the handler (``handler.lock``).
    2. ``handler.emit(record)``
    3. Release the lock.

For each emitted record above in ``handler.emit(record)``:
1. Call ``handler.format(record)`` to get the message
2. Send message out to target (this is custom depending on the handler
   implementation itself)

### Flow [Adapter version]

The adapter is a thin wrapper around its public ``logger`` attribute.

Assuming:

```python
extra = {"a": "b"}
adapter = logging.LoggerAdapter(logging.getLogger("logger"), extra=extra)
```

1. ``adapter.debug("Test")``
    1. ``adapter.log(DEBUG, "Test")``
2. ``adapter.log(level, msg, *args, **kwargs)``
    1. Ensure ``adapter.logger.isEnabledFor(DEBUG)``
    2. Call ``adapter.process(DEBUG, "Test")``
3. ``adapter.process(DEBUG, "Test", *args, **kwargs)``
    1. Attach ``kwargs["extra"] = self.extra``
4. Pass on the message and args to the original logger instance.
    1. ``self.logger.log(level, msg, *args, **kwargs)``

### Last Resort

If no other handlers are found in a log message chain, a "last resort" logger
is used.  

This is by default configured at the ``WARNING`` level.

The log level number requirement still applies for the last resort logger.

### Creating a log record

``logger.makeRecord`` takes care of creating a record.

Under the hood, log records are created through a factory
``logging.getLogRecordFactory()``, which can be set with
``logging.setLogRecordFactory(factory)``.

This factory (as of the time of writing) should have arguments:
```python
_logRecordFactory(
    name,
    level,
    filename,
    line_number,
    msg,
    args,
    exception_info,
    function,
    stack_info,
)
```

## Shutdown

There's an ``atexit`` hook which will look through every handler ever defined
(well, weakrefs of handlers, stashed in ``logging._handlerList``) and call the
following:

1. If the weakref is invalid, stop
2. Call the following, ignoring OSError/ValueError:
    1. ``handler.acquire()``
    2. ``handler.flush()``
    3. ``handler.close()``
4. And finally, ``handler.release()``

## Root Logger

Helper functions for using the root logger are there, but you really shouldn't
be using them (in my opinion).

There is at least one upside to using them, an application without logging
setup may still see the messages due to a check for handlers on the root logger
in these helpers.

``logging.info(...)`` -> ``basicConfig`` + ``root.info(...)``

``logging.debug(...)`` -> ``basicConfig`` + ``root.debug(...)``

``logging.warning(...)`` -> ``basicConfig`` + ``root.warning(...)``

``logging.error(...)`` -> ``basicConfig`` + ``root.error(...)``

``logging.exception(...)`` -> ``basicConfig`` + ``root.exception(...)``

``logging.critical(...)`` -> ``basicConfig`` + ``root.critical(...)``

## Capturing warnings

Warnings can be captured in the logging system by way of:

```python
logging.captureWarnings(True)
```

This (surprisingly, to me) monkey-patches ``warnings.showwarning`` to
redirect it to ``logging._showwarning``.  This formats the warning
and routes it to ``logging.getLogger("py.warnings").warning()``.


## basicConfig

This is used pretty much everywhere for simple configuration of logging.

As someone clearly interested in the Python logging system, it's worth reading
through the docstring once to see what it's actually doing.

>   Do basic configuration for the logging system.
>
>   This function does nothing if the root logger already has handlers
>   configured, unless the keyword argument *force* is set to ``True``.
>   It is a convenience method intended for use by simple scripts
>   to do one-shot configuration of the logging package.
>
>   The default behaviour is to create a StreamHandler which writes to
>   sys.stderr, set a formatter using the BASIC_FORMAT format string, and
>   add the handler to the root logger.
>
>   A number of optional keyword arguments may be specified, which can alter
>   the default behaviour.

Also, you can 

1. Specify ``filename``/``filemode`` to quickly create a ``FileHandler``
2. Configure formatting by way of ``format``/``datefmt``/``style`` and 
   ``encoding``
3. Configure the root logger level (``level``)
4. Redirect the default ``stderr`` stream (``stream``)
5. Pass in already-created handlers (``handlers = [...]``)


## Formatting

TODO

