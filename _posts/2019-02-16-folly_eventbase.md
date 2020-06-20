---
layout: blog
title: Studying folly EventBase from a test example 
date: 2019-02-16
tags: [dev]

---
folly [EventBase](https://github.com/facebook/folly/blob/master/folly/io/async/README.md) is an event handling module built on libevent. It is extremely useful in asynchronous and network programming. In this article, we start from the TestCase [EventBaseTest.ReadEvent](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/test/EventBaseTest.cpp#L188). The test creates a socket pair. On one side, it schedules two events that write byte streams to the socket. On the other side, it sets up a listener for events and read bytes from the socket. All operations share one event base.


```                                                                                             
                                                                           +------------+
                                                                +--------->| event_base |
                                                                |          +------------+
+--------------+               +----------+               +-----------+           |
| write event  |-->-write to-->|socket[1] |--local comm-->| socket[0] |        trigger
+--------------+               +----------+               +-----------+           |
        ^                                                       ^                 v
        |                                                       |            +---------------+
    timeout                                                     +-read from--| event handler |
        |                                                                    +---------------+
+--------------+                                                              
| event_base   |                                               
+--------------+
```

## Debug the code
As there are many asynchronous events and callbacks, it is really hard to catch the code path by barely reading the code. I set two break points to demystify `folly::EventBase`.

```
(gdb) info b
Num     Type           Disp Enb Address            What
4       breakpoint     keep y   0x0000000000459dc6 in ScheduledEvent::perform(int) at folly/io/async/test/EventBaseTest.cpp:120
        breakpoint already hit 1 time
7       breakpoint     keep y   0x00000000004cd315 in TestHandler::handlerReady(unsigned short) at folly/io/async/test/EventBaseTest.cpp:150
        breakpoint already hit 1 time
```

## Schedule events

`ScheduledEvent::perform` is hit first. Below is the callstack.
```
Breakpoint 4, ScheduledEvent::perform (this=0x7fffecd20440, fd=8) at folly/io/async/test/EventBaseTest.cpp:120
120         if (events & EventHandler::READ) {
(gdb) where
#0  ScheduledEvent::perform (this=0x7fffecd20440, fd=8) at folly/io/async/test/EventBaseTest.cpp:120
#1  0x0000000000508b9d in std::__invoke_impl<void, void (ScheduledEvent::*&)(int), ScheduledEvent*&, int&> (__f=
    @0x603000004c90: (void (ScheduledEvent::*)(ScheduledEvent * const, int)) 0x459db0 <ScheduledEvent::perform(int)>, __t=@0x603000004ca8: 0x7fffecd20440,
    __args=@0x603000004ca0: 8) at ../fbcode/third-party-buck/platform007/build/libgcc/include/c++/7.3.0/bits/invoke.h:73
#2  0x0000000000508987 in std::__invoke<void (ScheduledEvent::*&)(int), ScheduledEvent*&, int&> (__fn=
    @0x603000004c90: (void (ScheduledEvent::*)(ScheduledEvent * const, int)) 0x459db0 <ScheduledEvent::perform(int)>, __args=@0x603000004ca0: 8,
    __args=@0x603000004ca0: 8) at ../fbcode/third-party-buck/platform007/build/libgcc/include/c++/7.3.0/bits/invoke.h:95
#3  0x00000000005088d0 in std::_Bind<void (ScheduledEvent::*(ScheduledEvent*, int))(int)>::__call<void, , 0ul, 1ul>(std::tuple<>&&, std::_Index_tuple<0ul, 1ul>) (this=0x603000004c90, __args=...) at ../fbcode/third-party-buck/platform007/build/libgcc/include/c++/7.3.0/functional:467
#4  0x0000000000508725 in std::_Bind<void (ScheduledEvent::*(ScheduledEvent*, int))(int)>::operator()<, void>() (this=0x603000004c90)
    at ../fbcode/third-party-buck/platform007/build/libgcc/include/c++/7.3.0/functional:549
#5  0x00000000005081e0 in folly::detail::function::FunctionTraits<void ()>::callBig<std::_Bind<void (ScheduledEvent::*(ScheduledEvent*, int))(int)> >(folly::detail::function::Data&) (p=...) at buck-out/dev/gen/folly/function#header-mode-symlink-tree-with-header-map,headers/folly/Function.h:367
#6  0x00007ffff7f4c20a in folly::detail::function::FunctionTraits<void ()>::operator()() (this=0x611000000ea0)
    at buck-out/dev/gen/folly/function#header-mode-symlink-tree-with-header-map,headers/folly/Function.h:376
#7  0x00007ffff7fa918c in folly::TimeoutManager::CobTimeouts::CobTimeout::timeoutExpired (this=0x611000000e00) at folly/io/async/TimeoutManager.cpp:41
#8  0x00007ffff7f34c7b in folly::AsyncTimeout::libeventCallback (fd=-1, events=1, arg=0x611000000e00) at folly/io/async/AsyncTimeout.cpp:153
#9  0x00007ffff57c1c52 in event_process_active (base=<optimized out>) at event.c:390
#10 event_base_loop (base=0x61b000000080, flags=<optimized out>) at event.c:532
#11 0x00007ffff7f478ba in folly::EventBase::loopBody (this=0x7fffecd20020, flags=0, ignoreKeepAlive=false) at folly/io/async/EventBase.cpp:335
#12 0x00007ffff7f4665b in folly::EventBase::loop (this=0x7fffecd20020) at folly/io/async/EventBase.cpp:261
#13 0x000000000045afe3 in EventBaseTest_ReadEvent_Test::TestBody (this=0x602000001b90) at folly/io/async/test/EventBaseTest.cpp:206
...
#24 0x00007ffff7ff12d8 in main (argc=1, argv=0x7fffffffdd18) at common/gtest/LightMain.cpp:19
...
```

From the callstack we can see at the root is `EventBase::loop`. Some important classes with their methods are in the call chain from #13 upwards to #0. Let's spend sometime investigating them.  

[`class EventBase`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventBase.h#L128) inherits from five classes. One parent [`TimeoutManager`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/TimeoutManager.h#L33) is related to the test. `TimeoutManger` has a member variable [`std::unique_ptr<CobTimeouts> cobTimeouts_`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/TimeoutManager.h#L109) that stores a list of [`CobTimeout`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/TimeoutManager.cpp#L68)s. `CobTimeout` is a subclass of [`AsyncTimeout`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/AsyncTimeout.h#L81) and implements the function `timeoutExpired` which triggers the callback function stored in `CobTimeout::cob_`. `AsyncTimeout` has a member variable `TimeoutManager* timeoutManager_`, which is used to schedule timeout events.


```
                                                                    +---------------------+
                                                                    | AsyncTimeout        |
                                                                    +---------------------+
                    +-----------------------------------------------| timeoutManager_     |
                    |                                               | event_              |
+------------------------------+                                    +---------------------+
|  TimeoutManager              |                                            ^
+------------------------------+             +---------------+              |
| cobTimeouts_                 |-------------| CobTimeouts   |              |
+------------------------------+             +---------------+       +------------+
           ^                                 | list          |-------| CobTimeout |
           |                                 +---------------+       +------------+
+------------------------------+                                     | cob_       |
|  EventBase                   |                                     +------------+
+------------------------------+
| event_base* evb_             |
+------------------------------+
```
A simple UML like diagram of the class relationship.

How does the test hook `scheduledEvent::perform` up to the `eventBase`? Go back to the test and look at 
```
202 scheduleEvents(&eb, sp[1], events);
```
and its implementation 
```
void scheduleEvents(EventBase* eventBase, int fd, ScheduledEvent* events) {
  for (ScheduledEvent* ev = events; ev->milliseconds > 0; ++ev) {
    eventBase->tryRunAfterDelay(
        std::bind(&ScheduledEvent::perform, ev, fd), ev->milliseconds);
  }
}
```
[`tryRunAferDelay`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/TimeoutManager.cpp#L91) is defined in EventBase's parent class `TimeoutManager`. The function `ScheduledEvent::perform` with the `fd` parameter bound by `socket[1]` is pass to `CobTimeout` as the callback function `cob_` and the `eventBase` itself is passed to `CobTimeout` as the `timeoutManager_`. For each `ScheduledEvent`, a `CobTimeout` is created and inserted to the list `cobTimeouts_`. The `CobTimeout` calls [`scheduleTimeout`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/AsyncTimeout.cpp#L87) meaning that a timeout event will happen in `milliseconds`. `scheduleTimeout` is implemented in EventBase (decleared in TimeoutManager. Remember EventBase inherits from TimeoutManager).

I would like to spend some time on the low level logic in `EventBase::scheduleTimeout` that interacts with our core library [libevent](https://libevent.org/). I believe this part is useful to understand how we hop to `AsyncTimeout::libeventCallback` from `EventBase::loopBody` (callstack #8~#11). `tryRunAfterDelay` creates CobTimeout (parent `AsyncTimeout`) for each `ScheduledEvent`. [AsyncTimeout's constructor](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/AsyncTimeout.cpp#L29), sets `event_` with the event type `EV_TIMEOUT` and callback function `&AsyncTimeout::libeventCallback` and calls `EventBase::attachTimeoutManager` which [associates its `event_base* evb_` to the `CobTimeout`'s `event_`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventBase.cpp#L734). Then come back to `EventBase::scheduleTimeout`, it [adds the timeout value to `CobTimeout::event_`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventBase.cpp#L760).

After the configuration, the timeout will expire in the specified milliseconds and the callback function `ScheduledEvent::perform` will be triggered. More specifically, `EventBase::loopBody` (callstack #11) calls `event_base_loop(evb_, EVLOOP_ONCE)` (callstack #10). The thread waits until the timeout fires, when the events we scheduled in `scheduleEvents` become active. `event_process_active` (callstack #9) calls the callback function [`AsyncTimeout::libeventCallback`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/AsyncTimeout.cpp#L137). Here are some nuances in the callback function.
```
void AsyncTimeout::libeventCallback(libevent_fd_t fd, short events, void* arg) {
  AsyncTimeout* timeout = reinterpret_cast<AsyncTimeout*>(arg);
  ...
  timeout->timeoutExpired();
}
```
First, it creates a new `AsyncTimeout timeout` from the last parameter `arg`. As I mentioned before, in AsyncTimeout's constructor `event_set`'s last parameter is exactly the `arg`. In the constructor, we pass `this` pointer to `arg`, so here `timeout` is the AsyncTimeout (CobTimeout) we created in `tryRunAfterDelay`. `timeout->timeoutExpired()` calls the `cob_` function, which is `ScheduledEvent::perform`. That is the whole logic for callstack (#8 ~ #0).

## Register the handler
After the event scheduling, the program hits the second breakpoint.
```
Breakpoint 7, TestHandler::handlerReady (this=0x7fffecd20310, events=2) at folly/io/async/test/EventBaseTest.cpp:150
150         ssize_t bytesRead = 0;
(gdb) where
#0  TestHandler::handlerReady (this=0x7fffecd20310, events=2) at folly/io/async/test/EventBaseTest.cpp:150
#1  0x00007ffff7f92dad in folly::EventHandler::libeventCallback (fd=7, events=2, arg=0x7fffecd20310) at folly/io/async/EventHandler.cpp:161
#2  0x00007ffff57c1c52 in event_process_active (base=<optimized out>) at event.c:390
#3  event_base_loop (base=0x61b000000080, flags=<optimized out>) at event.c:532
#4  0x00007ffff7f478ba in folly::EventBase::loopBody (this=0x7fffecd20020, flags=0, ignoreKeepAlive=false) at folly/io/async/EventBase.cpp:335
#5  0x00007ffff7f4665b in folly::EventBase::loop (this=0x7fffecd20020) at folly/io/async/EventBase.cpp:261
#6  0x000000000045afe3 in EventBaseTest_ReadEvent_Test::TestBody (this=0x602000001b90) at folly/io/async/test/EventBaseTest.cpp:206
...
#17 0x00007ffff7ff12d8 in main (argc=1, argv=0x7fffffffdd18) at common/gtest/LightMain.cpp:19
...
```
How is `TestHandler::handlerReady` linked to `EventBase`? Similar to `SecheduledEvent`, TestHandler (parent EventHandler) sets the event with callbacks in [`EventHandler::registerImpl`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventHandler.cpp#L62) with the callback function [`EventHandler::libeventCallback`](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventHandler.cpp#L148), in which `handler->handlerReady(uint16_t(events));` is executed. The handler is exactly the `TestHandler` object created in the test. That explains the callstack #0. The parameter `events`'s value is 2 (`EV_READ`), consistent with the stream flow. 

## EventBase Loop
The main logic of `EventBase::loop` is a big [`while` loop](https://github.com/facebook/folly/blob/08a5c35995f27de4b6b05fff784bfea7a37e118b/folly/io/async/EventBase.cpp#L316). The snippet below contains important statements in this loop.

```
while (!stop_.load(std::memory_order_relaxed)) {
    ...
    LoopCallbackList callbacks;
    callbacks.swap(runBeforeLoopCallbacks_);

    while (!callbacks.empty()) {
      auto* item = &callbacks.front();
      callbacks.pop_front();
      item->runLoopCallback();
    }

    ...
    if (blocking && loopCallbacks_.empty()) {
      res = event_base_loop(evb_, EVLOOP_ONCE);
    } else {
      res = event_base_loop(evb_, EVLOOP_ONCE | EVLOOP_NONBLOCK);
    }

    ranLoopCallbacks = runLoopCallbacks();
    if (res != 0) {
      if (getNotificationQueueSize() > 0) {
        fnRunner_->handlerReady(0);
      } else if (!ranLoopCallbacks) {
        break;
      }
    }
}
```
As test does not specify `runBeforeLoopCallbacks_` and `loopCallbacks_`, the only callback functions are `ScheduledEvent::perform` and `TestHandler::handlerReady` in `event_base_loop`.  There are two events for `ScheduledEvent` whilst only one event for `TestHandler`. The `event_base_loop` will get called three times.

1. Initially, there are two pending timeout events (ScheduledEvent) in the `event_base`.
1. First write event happens, `event_base_loop` gets called. As there is one pending event remaining, `res` is returned as 1. The loop continues.
1. The write event callback write to the socket, which is listened by `TestHandler`. One read event become active and `event_base_loop` gets called. `TestHandler::handlerReady` reads bytes from the socket.
1. Second write event happens, `event_base_loop` get called. As no event left, `res` is returned as 0. `while` loop terminates.

The PERSIST flag is not specified for `TestHandler`, meaning that `TestHandler` will only catch the bytes written by the first event. That explains why the testHandler has one log.


List of Structs and Functions in libevent
* `struct event`
* `struct event_base`
* [`event_set`](http://www.wangafu.net/~nickm/libevent-2.0/doxygen/html/event_8h.html#a44df7b40859b56f2c866adb02dabdd9e)
* [`event_base_set`](http://www.wangafu.net/~nickm/libevent-2.0/doxygen/html/event_8h.html#a44df7b40859b56f2c866adb02dabdd9e)
* [`event_add`](http://www.wangafu.net/~nickm/libevent-2.0/doxygen/html/event_8h.html#a44df7b40859b56f2c866adb02dabdd9e)
* [`event_baes_loop`](http://www.wangafu.net/~nickm/libevent-2.0/doxygen/html/event_8h.html#a44df7b40859b56f2c866adb02dabdd9e)

# References
1. [GitHub facebook/folly](https://github.com/facebook/folly)
1. [libevent](https://libevent.org/)
1. [libevent-book Ref4 Working with events](http://www.wangafu.net/~nickm/libevent-book/Ref4_event.html)
