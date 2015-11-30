Tango/PyTango 9 installation report
===================================

This report lists the different issues or warnings I ran into while installing Tango 9 and PyTango 9 on my local computer (64bits 14.04 Ubuntu).


Tango 9 Configuration
---------------------

    Version:                9.1.0
    Compiler:               gcc,g++
    OMNIORB PATH:           /usr/local
    OMNIORB VERSION:
    ZMQ PATH:               /usr
    ZMQ VERSION:            4.0.5
    JAVA PATH:              /usr/bin/java
    JAVA VERSION:           1.7.0_79
    MYSQL CLIENT LIB:        -lmysqlclient_r
    MYSQL VERSION:          5.5.43
    MYSQL CONNECTION:       OK


Tango build warnings
--------------------

Here is the 4 `unused-variable` warnings I got while building Tango 9:

```
ClassFactory.cpp:2:20: warning: ‘RcsId’ defined but not used [-Wunused-variable]
 static const char *RcsId = "$Header$";
                    ^
DataBaseStateMachine.cpp:2:20: warning: ‘RcsId’ defined but not used [-Wunused-variable]
 static const char *RcsId = "$Id: DataBaseStateMachine.cpp 27961 2015-05-11 11:04:49Z taurel $";
                    ^
main.cpp:2:20: warning: ‘RcsId’ defined but not used [-Wunused-variable]
 static const char *RcsId = "$Id: main.cpp 26081 2014-07-17 15:02:38Z taurel $";
                    ^
DataBaseUtils.cpp:1:20: warning: ‘RcsId’ defined but not used [-Wunused-variable]
 static const char *RcsId = "$Header$";
                    ^
```


Error while loading libzmq library
----------------------------------

After building and installing Tango 9, I tried to run the starter and ran onto the following error:

```
tango_admin: error while loading shared libraries: libzmq.so.3: cannot open shared object file: No such file or directory
```

That's because I installed libzmq3-4.0.5 (and libzmq3-dev-4.0.5) that are only providing the following file:

- /usr/lib/x86_64-linux-gnu/libzmq.so
- /usr/lib/x86_64-linux-gnu/libzmq.so.4
- /usr/lib/x86_64-linux-gnu/libzmq.so.4.0.0

So I had to run the following command to get it to work:

```bash
$ sudo ln -s /usr/lib/x86_64-linux-gnu/libzmq.so /usr/lib/x86_64-linux-gnu/libzmq.so.3
```


PyTango build warnings
----------------------

Here is the different warnings that I got while building PyTango 9 (144 total):

- x114: unused variable 'rc'
- x29: using deprecated NumPy API, disable it by #defining NPY_NO_DEPRECATED_API NPY_1_7_API_VERSION
- x1: 'auto_ptr' is deprecated (declared at /usr/include/c++/4.8/backward/auto_ptr.h:87)

See the full log [here](pytango9-warnings.log).


IPython warnings
----------------

After building and installing PyTango 9 in a virtualenv, I ran `itango` and got the following warnings:

- ShimWarning: The `IPython.config` package has been deprecated. You should import from traitlets.config instead.
- UserWarning: get_ipython_dir has moved to the IPython.paths module
- UserWarning: IPython.utils.traitlets has moved to a top-level traitlets package.
- ShimWarning: The `IPython.qt` package has been deprecated. You should import from qtconsole instead.

I fixed them by updating the corresponding imports. However it might break compatibility with older IPython versions.
In any case, the changes can be found in the following commit: [fix ipython warning](https://github.com/vxgmichel/PyTango/commit/ac6ce687f9d1c995794b3e9a27ebfc05a0bc272f).


Database hostname issue
-----------------------

`itango` also printed the following warning:

```
Could not access any Database. Make sure:
    - .tangorc, /etc/tangorc or TANGO_HOST environment is defined.
    - the Database DS is running
```

That's because something changed with the hostname resolution. That's my `tangorc` configuration:

```bash
$ cat /etc/tangorc
TANGO_HOST=localhost:10000
```

This is what I get with Tango 8:

```python
>>> PyTango.Database()
Database(vinmic-t440p, 10000)
>>> PyTango.Database('vinmic-t440p', 10000)
Database(vinmic-t440p, 10000)
```

And that's what I get with Tango 9:

```python
>>> PyTango.Database()
Database('vinmic-t440p.maxiv.lu.se', 10000)
>>> Database('vinmic-t440p.maxiv.lu.se', 10000)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
PyTango.ConnectionFailed: DevFailed[
DevError[
   desc = TRANSIENT CORBA system exception: TRANSIENT_ConnectFailed
 origin = Connection::connect
  reason = API_CorbaException
severity = ERR]

DevError[
    desc = Failed to connect to database on host vinmic-t440p.maxiv.lu.se with port 10000
  origin = Connection::connect
  reason = API_CantConnectToDatabase
severity = ERR]
]
```

Subscribing twice to the same attribute
---------------------------------------

Subscribing twice to the same attribute raises an exception in Tango 9:

```
DevError[
    desc = Failed to connect to event channel for device tango://vinmic-t440p.maxiv.lu.se:10000/sys/tg_test/1
           Corrupted internal map: event callback already exists. Please report bug!
  origin = EventConsumer::connect_event()
  reason = API_NotificationServiceFailed
severity = ERR]
```

This happens even if the proxies or callbacks are different objects.
The following example shows how to reproduce the bug:


```python
>>> import PyTango
>>> d1 = PyTango.DeviceProxy('sys/tg_test/1')
>>> def f(arg): print arg.attr_value.value
...
>>> d1.subscribe_event('State', PyTango.EventType.CHANGE_EVENT, f)
RUNNING
1
>>> d2 = PyTango.DeviceProxy('sys/tg_test/1')
>>> def g(arg): print arg.attr_value.value
...
>>> d2.subscribe_event('State', PyTango.EventType.CHANGE_EVENT, g)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/home/vinmic/.virtualenvs/tango9/local/lib/python2.7/site-packages/PyTango/green.py", line 173, in greener
    ret = submit(fn, self, *args, **kwargs)
  File "/home/vinmic/.virtualenvs/tango9/local/lib/python2.7/site-packages/PyTango/green.py", line 83, in synch_submit
    return get_synch_executor().submit(fn, *args, **kwargs)
  File "/home/vinmic/.virtualenvs/tango9/local/lib/python2.7/site-packages/PyTango/green.py", line 75, in submit
    return fn(*args, **kwargs)
  File "/home/vinmic/.virtualenvs/tango9/local/lib/python2.7/site-packages/PyTango/device_proxy.py", line 853, in __DeviceProxy__subscribe_event
    event_id = self.__subscribe_event(attr_name, event_type, cb, filters, stateless, extract_as)
PyTango.EventSystemFailed: DevFailed[
DevError[
    desc = Failed to connect to event channel for device tango://vinmic-t440p.maxiv.lu.se:10000/sys/tg_test/1
           Corrupted internal map: event callback already exists. Please report bug!
  origin = EventConsumer::connect_event()
  reason = API_NotificationServiceFailed
severity = ERR]
]
```

In Tango 8, this last line would simply print `RUNNING` and return `2`.
