##ioloop分析

######咱们先来简单说一下ioloop的源码
```
while True:
    poll_timeout = 3600.0
    
    # Prevent IO event starvation by delaying new callbacks
    # to the next iteration of the event loop.
    with self._callback_lock:
        ＃上次循环的回调列表
    	callbacks = self._callbacks
    	self._callbacks = []
    for callback in callbacks:
        ＃执行遗留回调
    	self._run_callback(callback)
    
    if self._timeouts:
    	now = self.time()
    	while self._timeouts:
    	    ＃超时回调
    		if self._timeouts[0].callback is None:
    			# 最小堆维护超时事件
    			heapq.heappop(self._timeouts)
    		elif self._timeouts[0].deadline <= now:
    			timeout = heapq.heappop(self._timeouts)
    			self._run_callback(timeout.callback)
    		else:
    			seconds = self._timeouts[0].deadline - now
    			poll_timeout = min(seconds, poll_timeout)
    			break
    
    if self._callbacks:
    	# If any callbacks or timeouts called add_callback,
    	# we don't want to wait in poll() before we run them.
    	poll_timeout = 0.0
    
    if not self._running:
    	break
    
    if self._blocking_signal_threshold is not None:
    	# clear alarm so it doesn't fire while poll is waiting for
    	# events.
    	signal.setitimer(signal.ITIMER_REAL, 0, 0)
    
    try:
        ＃这里的poll就是epoll，当有事件发生，就会返回，详情参照tornado里e＃poll的代码
    	event_pairs = self._impl.poll(poll_timeout)
    except Exception as e:
    	if (getattr(e, 'errno', None) == errno.EINTR or
    		(isinstance(getattr(e, 'args', None), tuple) and
    		 len(e.args) == 2 and e.args[0] == errno.EINTR)):
    		continue
    	else:
    		raise
    
    if self._blocking_signal_threshold is not None:
    	signal.setitimer(signal.ITIMER_REAL,
    					 self._blocking_signal_threshold, 0)
    
    ＃如果有事件发生，添加事件，
    self._events.update(event_pairs)
    while self._events:
    	fd, events = self._events.popitem()
    	try:
    	    ＃根据fd找到对应的回调函数，
    		self._handlers[fd](fd, events)
    	except (OSError, IOError) as e:
    		if e.args[0] == errno.EPIPE:
    			# Happens when the client closes the connection
    			pass
    		else:
    			app_log.error("Exception in I/O handler for fd %s",
    						  fd, exc_info=True)
    	except Exception:
    		app_log.error("Exception in I/O handler for fd %s",
    					  fd, exc_info=True)
    					  
    					  ```
    					  
#####  简单来说一下流程，首先执行上次循环的回调列表，然后调用epoll等待事件的发生，根据fd取出对应的回调函数，然后执行。IOLoop基本是个事件循环，因此它总是被其它模块所调用。


##### 咱们来看一下ioloop的start方法，start 方法中主要分三个部分：一个部分是对超时的相关处理；一部分是 epoll 事件通知阻塞、接收；一部分是对 epoll 返回I/O事件的处理。

1.超时的处理，是用一个最小堆来维护每个回调函数的超时时间，如果超时，取出对应的回调，如果没有则重新设置poll_timeout的值

2.通过 self._impl.poll(poll_timeout) 进行事件阻塞，当有事件通知或超时时 poll 返回特定的 event_pairs，这个上面也说过了。
