---
title: 多线程原理--GCD源码分析
date: 2019-11-03 12:39:56
author: Vincent
top: true
categories: 
- 底层原理
- 源码分析
tags: 
- GCD
- 源码分析
- 底层原理
---


![多线程原理--GCD源码分析](https://upload-images.jianshu.io/upload_images/5741330-0205b4b0f9893149.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

阅读源码是枯燥的，可能暂时对我们的工作没什么帮助，现在但是作为一个有一定开发经验的开发人员而言，这一步是必须要走的；可能是受到了身边同事、同行的影响，看别人在读源码也跟着读源码，或者是开发中遇到了瓶颈，亦或者是开发不再局限于业务的开发，需要架构设计、设计模式以及数据结构和算法等需要阅读源码等等，一般开始的时候真的很难读懂，看的头大，但是当我们用尽办法研究通后，那个时候真的很爽。我们不再只是知道这样写，我们可以知其然知其所以然，知道有些函数是做什么的，知道其底层原理是怎样的，比如同样实现一个功能可以用很多种方法，我们知道这些方法底层原理后可以知道这些方法的本质区别，我们可以通过阅读源码学习到一些更好的设计思想、有更好的问题解决方案等等，也可以锻炼我们的耐心和毅力，阅读源码对我们来说真的受益无穷。

> 如果还不是很了解GCD，可以先简单了解一下GCD：[多线程原理--了解GCD](https://www.jianshu.com/p/acc6e7bd6f10)，接下来开始分析当前最新版本的源码：[libdispatch-1008.200.78.tar.gz](https://opensource.apple.com/tarballs/libdispatch/libdispatch-1008.200.78.tar.gz)，建议去获取最新版本GCD源码：[opensource源](https://opensource.apple.com/tarballs/)或者[github源](https://github.com/apple)。


#### 创建队列dispatch_queue_create

我们可以探索下串行和并发队列的区别。
首先跟着创建队列函数`dispatch_queue_create`进入源码，除了我们赋值的label和attr，系统还将tq赋值`DISPATCH_TARGET_QUEUE_DEFAULT `，legacy 赋值为`true`传给`dispatch_queue_create_with_target`，其内部首先通过`_dispatch_queue_attr_to_info`和我们传进来的attr来初始化`dispatch_queue_attr_info_t`。
```
dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
	return _dispatch_lane_create_with_target(label, attr,
			DISPATCH_TARGET_QUEUE_DEFAULT, true);
}

---------------------

static dispatch_queue_t
_dispatch_lane_create_with_target(const char *label, dispatch_queue_attr_t dqa,
		dispatch_queue_t tq, bool legacy)
{
	dispatch_queue_attr_info_t dqai = _dispatch_queue_attr_to_info(dqa);
  // ...省略N行代码--部分代码
	const void *vtable;
	dispatch_queue_flags_t dqf = legacy ? DQF_MUTABLE : 0;
	if (dqai.dqai_concurrent) {
		vtable = DISPATCH_VTABLE(queue_concurrent);
	} else {
		vtable = DISPATCH_VTABLE(queue_serial);
	}

    dispatch_lane_t dq = _dispatch_object_alloc(vtable,
			sizeof(struct dispatch_lane_s));
	_dispatch_queue_init(dq, dqf, dqai.dqai_concurrent ?
			DISPATCH_QUEUE_WIDTH_MAX : 1, DISPATCH_QUEUE_ROLE_INNER |
			(dqai.dqai_inactive ? DISPATCH_QUEUE_INACTIVE : 0));
	dq->dq_label = label;
	dq->dq_priority = _dispatch_priority_make((dispatch_qos_t)dqai.dqai_qos,
			dqai.dqai_relpri);
    // 自定义的queue的目标队列是root队列
	dq->do_targetq = tq;
	_dispatch_object_debug(dq, "%s", __func__);
	return _dispatch_trace_queue_create(dq)._dq;
```

再次通过全局搜索`_dispatch_queue_attr_to_info`，来查看`_dispatch_queue_attr_to_info`内部的实现。
```
dispatch_queue_attr_info_t
_dispatch_queue_attr_to_info(dispatch_queue_attr_t dqa)
{
	dispatch_queue_attr_info_t dqai = { };

	if (!dqa) return dqai;

#if DISPATCH_VARIANT_STATIC
	if (dqa == &_dispatch_queue_attr_concurrent) {
		dqai.dqai_concurrent = true;
		return dqai;
	}
#endif

	if (dqa < _dispatch_queue_attrs ||
			dqa >= &_dispatch_queue_attrs[DISPATCH_QUEUE_ATTR_COUNT]) {
		DISPATCH_CLIENT_CRASH(dqa->do_vtable, "Invalid queue attribute");
	}

	size_t idx = (size_t)(dqa - _dispatch_queue_attrs);

	dqai.dqai_inactive = (idx % DISPATCH_QUEUE_ATTR_INACTIVE_COUNT);
	idx /= DISPATCH_QUEUE_ATTR_INACTIVE_COUNT;

	dqai.dqai_concurrent = !(idx % DISPATCH_QUEUE_ATTR_CONCURRENCY_COUNT);
	idx /= DISPATCH_QUEUE_ATTR_CONCURRENCY_COUNT;

	dqai.dqai_relpri = -(idx % DISPATCH_QUEUE_ATTR_PRIO_COUNT);
	idx /= DISPATCH_QUEUE_ATTR_PRIO_COUNT;

	dqai.dqai_qos = idx % DISPATCH_QUEUE_ATTR_QOS_COUNT;
	idx /= DISPATCH_QUEUE_ATTR_QOS_COUNT;

	dqai.dqai_autorelease_frequency =
			idx % DISPATCH_QUEUE_ATTR_AUTORELEASE_FREQUENCY_COUNT;
	idx /= DISPATCH_QUEUE_ATTR_AUTORELEASE_FREQUENCY_COUNT;

	dqai.dqai_overcommit = idx % DISPATCH_QUEUE_ATTR_OVERCOMMIT_COUNT;
	idx /= DISPATCH_QUEUE_ATTR_OVERCOMMIT_COUNT;

	return dqai;
}
```
`_dispatch_queue_attr_to_info`方法内部首先判断我们传进来的dqa是否为空，如果为空则直接返回空结构体，也就是我们所说的串行队列（串行队列我们一般传DISPATCH_QUEUE_SERIAL或者NULL，其实DISPATCH_QUEUE_SERIAL的宏定义就是NULL）。如果不为空，则进入苹果的算法，通过结构体位域来设置dqai的属性并返回该结构体`dispatch_queue_attr_info_t`。结构体：
```
typedef struct dispatch_queue_attr_info_s {
	dispatch_qos_t dqai_qos : 8;
	int      dqai_relpri : 8;
	uint16_t dqai_overcommit:2;
	uint16_t dqai_autorelease_frequency:2;
	uint16_t dqai_concurrent:1;
	uint16_t dqai_inactive:1;
} dispatch_queue_attr_info_t;
```
再次回到`_dispatch_lane_create_with_target `内部，接下来出来overcommit（如果是串行队列的话默认是开启的，并行是关闭的）、`_dispatch_get_root_queue `获取一个管理自己队列的root队列，每一个优先级都有对应的root队列，每一个优先级又分为是不是可以过载的队列。再通过`dqai.dqai_concurrent`来区分并发和串行，`DISPATCH_VTABLE`内部利用`OS_dispatch_##name##_class`生成相应的class保存到结构体`dispatch_queue_t`内的do_vtable变量，接下来开辟内存`_dispatch_object_alloc `、构造方法`_dispatch_queue_init`这里的第三个参数判断是否并行队列，如果不是则最多开辟一条线程，如果是并行队列则最多可以开辟DISPATCH_QUEUE_WIDTH_FULL(0x1000) - 2条，也就是0xffe换算成10进制就是4094条线程，接下来就是设置dq的dq_label、dq_priority等属性，最后返回`_dispatch_trace_queue_create(dq)._dq`。进入其内部再次返回`_dispatch_introspection_queue_create(dqu)`，直到进入`_dispatch_introspection_queue_create_hook`内部的`dispatch_introspection_queue_get_info`返回串行或者并行的结构体用来保存关于队列的信息。`dispatch_introspection_queue_s`：
```
dispatch_introspection_queue_s diq = {
		.queue = dq->_as_dq,
		.target_queue = dq->do_targetq,
		.label = dq->dq_label,
		.serialnum = dq->dq_serialnum,
		.width = dq->dq_width,
		.suspend_count = _dq_state_suspend_cnt(dq_state) + dq->dq_side_suspend_cnt,
		.enqueued = _dq_state_is_enqueued(dq_state) && !global,
		.barrier = _dq_state_is_in_barrier(dq_state) && !global,
		.draining = (dq->dq_items_head == (void*)~0ul) ||
				(!dq->dq_items_head && dq->dq_items_tail),
		.global = global,
		.main = dx_type(dq) == DISPATCH_QUEUE_MAIN_TYPE,
	};

---------

dispatch_introspection_queue_s diq = {
		.queue = dwl->_as_dq,
		.target_queue = dwl->do_targetq,
		.label = dwl->dq_label,
		.serialnum = dwl->dq_serialnum,
		.width = 1,
		.suspend_count = 0,
		.enqueued = _dq_state_is_enqueued(dq_state),
		.barrier = _dq_state_is_in_barrier(dq_state),
		.draining = 0,
		.global = 0,
		.main = 0,
	};
```

对于`dispatch_get_global_queue`从底层`_dispatch_get_root_queue `中取得合适的队列，其可以开辟DISPATCH_QUEUE_WIDTH_FULL(0x1000) - 1条线程，也就是0xfff，并且从`dispatch_queue_s _dispatch_root_queues[]`全局属性里面存放各种global_queue；而对于`dispatch_get_main_queue `则是通过`DISPATCH_GLOBAL_OBJECT(dispatch_queue_main_t, _dispatch_main_q);`返回，通过全局搜索
```
DISPATCH_DECL_SUBCLASS(dispatch_queue_main, dispatch_queue_serial);
#define OS_OBJECT_DECL_SUBCLASS(name, super) \
		OS_OBJECT_DECL_IMPL(name, <OS_OBJECT_CLASS(super)>)
```
可以发现`dispatch_queue_main`就是串行`dispatch_queue_serial`的子类，线程的width同样是0x1，也就是只有1条。

#### 同步dispatch_sync
接下来研究一下同步函数`dispatch_sync`，查看其源码进入内部方法`_dispatch_sync_f`，再次进入`_dispatch_sync_f_inline`内部：

```
DISPATCH_NOINLINE
static void
_dispatch_sync_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func,
		uintptr_t dc_flags)
{
	_dispatch_sync_f_inline(dq, ctxt, func, dc_flags);
}

---------

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_sync_f_inline(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	if (likely(dq->dq_width == 1)) {
		return _dispatch_barrier_sync_f(dq, ctxt, func, dc_flags);
	}

	if (unlikely(dx_metatype(dq) != _DISPATCH_LANE_TYPE)) {
		DISPATCH_CLIENT_CRASH(0, "Queue type doesn't support dispatch_sync");
	}

	dispatch_lane_t dl = upcast(dq)._dl;
	// Global concurrent queues and queues bound to non-dispatch threads
	// always fall into the slow case, see DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE
	if (unlikely(!_dispatch_queue_try_reserve_sync_width(dl))) {
		return _dispatch_sync_f_slow(dl, ctxt, func, 0, dl, dc_flags);
	}

	if (unlikely(dq->do_targetq->do_targetq)) {
		return _dispatch_sync_recurse(dl, ctxt, func, dc_flags);
	}
	_dispatch_introspection_sync_begin(dl);
	_dispatch_sync_invoke_and_complete(dl, ctxt, func DISPATCH_TRACE_ARG(
			_dispatch_trace_item_sync_push_pop(dq, ctxt, func, dc_flags)));
}
```

首先判断dq_width是否等于1,，也就是当前队列是否是串行队列，如果是则执行`_dispatch_barrier_sync_f `，经过一系列的嵌套最终走到`_dispatch_barrier_sync_f_inline`，`_dispatch_barrier_sync_f_inline `内部先通过`_dispatch_thread_port`获取当前线程ID，进入`_dispatch_queue_try_acquire_barrier_sync `判断线程状态，进入内部`_dispatch_queue_try_acquire_barrier_sync_and_suspend`，在这里会通过`os_atomic_rmw_loop2o`来获取当前队列依赖线程的状态信息；如果判断当前队列是全局并行队列或者绑定的是非调度线程的队列会直接进入if判断内执行`_dispatch_sync_f_slow `，在`_dispatch_sync_f_slow `内部会执行同步等待`__DISPATCH_WAIT_FOR_QUEUE__`，这里涉及到死锁的问题，其内部会将等待的队列`_dispatch_wait_prepare`和当前调度的队列进行对比`_dq_state_drain_locked_by(dq_state, dsc->dsc_waiter)`，如果相同则直接抛出crash："dispatch_sync called on queue " "already owned by current thread"；如果没有产生死锁，最后执行`_dispatch_sync_invoke_and_complete_recurse `，其内部先执行`_dispatch_thread_frame_push`把任务压栈到队列后再执行func（block任务）后mach底层通过hook函数来监听complete，再`_dispatch_thread_frame_pop`把任务pop出去，这也就是为什么同步并发会顺序执行的原因。

`_dispatch_barrier_sync_f_inline`：
```
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_barrier_sync_f_inline(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	dispatch_tid tid = _dispatch_tid_self();

	if (unlikely(dx_metatype(dq) != _DISPATCH_LANE_TYPE)) {
		DISPATCH_CLIENT_CRASH(0, "Queue type doesn't support dispatch_sync");
	}

	dispatch_lane_t dl = upcast(dq)._dl;
	// The more correct thing to do would be to merge the qos of the thread
	// that just acquired the barrier lock into the queue state.
	//
	// However this is too expensive for the fast path, so skip doing it.
	// The chosen tradeoff is that if an enqueue on a lower priority thread
	// contends with this fast path, this thread may receive a useless override.
	//
	// Global concurrent queues and queues bound to non-dispatch threads
	// always fall into the slow case, see DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE
	
	if (unlikely(!_dispatch_queue_try_acquire_barrier_sync(dl, tid))) {
		return _dispatch_sync_f_slow(dl, ctxt, func, DC_FLAG_BARRIER, dl,
				DC_FLAG_BARRIER | dc_flags);
	}

	if (unlikely(dl->do_targetq->do_targetq)) {
		return _dispatch_sync_recurse(dl, ctxt, func,
				DC_FLAG_BARRIER | dc_flags);
	}
	_dispatch_introspection_sync_begin(dl);
	_dispatch_lane_barrier_sync_invoke_and_complete(dl, ctxt, func
			DISPATCH_TRACE_ARG(_dispatch_trace_item_sync_push_pop(
					dq, ctxt, func, dc_flags | DC_FLAG_BARRIER)));
}

DISPATCH_NOINLINE
static void
_dispatch_barrier_sync_f(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	_dispatch_barrier_sync_f_inline(dq, ctxt, func, dc_flags);
}
```

如果是全局并行队列或者绑定的是非调度线程的队列会直接进入`_dispatch_sync_f_slow `和上述逻辑相同。如果是加入栅栏函数的则开始验证target是否存在，`_dispatch_sync_recurse`内递归`_dispatch_sync_wait`进行查找target，直到找到target后执行`_dispatch_sync_invoke_and_complete_recurse`完成回调。


#### 异步dispatch_async

进入`dispatch_async`源码内部，先进行了初始化操作：
```
void
dispatch_async(dispatch_queue_t dq, dispatch_block_t work)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags);
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}
```

进入`_dispatch_continuation_init`内部将`dispatch_async`的block任务重新赋值给func并保持为dc的dc_func属性。接下来执行`_dispatch_continuation_async `，最后进入`_dispatch_continuation_async`内部的`dx_push`，通过宏定义`#define dx_push(x, y, z) dx_vtable(x)->dq_push(x, y, z)`，我们选择进入全局并发队列`_dispatch_root_queue_push`，最终进入`_dispatch_root_queue_poke_slow `：
```
static void
_dispatch_root_queue_poke_slow(dispatch_queue_global_t dq, int n, int floor)
{
	int remaining = n;
	int r = ENOSYS;

	_dispatch_root_queues_init();
	_dispatch_debug_root_queue(dq, __func__);
	_dispatch_trace_runtime_event(worker_request, dq, (uint64_t)n);

#if !DISPATCH_USE_INTERNAL_WORKQUEUE
#if DISPATCH_USE_PTHREAD_ROOT_QUEUES
	if (dx_type(dq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE)
#endif
	{
		_dispatch_root_queue_debug("requesting new worker thread for global "
				"queue: %p", dq);
		r = _pthread_workqueue_addthreads(remaining,
				_dispatch_priority_to_pp_prefer_fallback(dq->dq_priority));
		(void)dispatch_assume_zero(r);
		return;
	}
#endif // !DISPATCH_USE_INTERNAL_WORKQUEUE
#if DISPATCH_USE_PTHREAD_POOL
	dispatch_pthread_root_queue_context_t pqc = dq->do_ctxt;
	if (likely(pqc->dpq_thread_mediator.do_vtable)) {
		while (dispatch_semaphore_signal(&pqc->dpq_thread_mediator)) {
			_dispatch_root_queue_debug("signaled sleeping worker for "
					"global queue: %p", dq);
			if (!--remaining) {
				return;
			}
		}
	}

	bool overcommit = dq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
	if (overcommit) {
		os_atomic_add2o(dq, dgq_pending, remaining, relaxed);
	} else {
		if (!os_atomic_cmpxchg2o(dq, dgq_pending, 0, remaining, relaxed)) {
			_dispatch_root_queue_debug("worker thread request still pending for "
					"global queue: %p", dq);
			return;
		}
	}

	int can_request, t_count;
	// seq_cst with atomic store to tail <rdar://problem/16932833>
	t_count = os_atomic_load2o(dq, dgq_thread_pool_size, ordered);
	do {
		can_request = t_count < floor ? 0 : t_count - floor;
		if (remaining > can_request) {
			_dispatch_root_queue_debug("pthread pool reducing request from %d to %d",
					remaining, can_request);
			os_atomic_sub2o(dq, dgq_pending, remaining - can_request, relaxed);
			remaining = can_request;
		}
		if (remaining == 0) {
			_dispatch_root_queue_debug("pthread pool is full for root queue: "
					"%p", dq);
			return;
		}
	} while (!os_atomic_cmpxchgvw2o(dq, dgq_thread_pool_size, t_count,
			t_count - remaining, &t_count, acquire));

	pthread_attr_t *attr = &pqc->dpq_thread_attr;
	pthread_t tid, *pthr = &tid;
#if DISPATCH_USE_MGR_THREAD && DISPATCH_USE_PTHREAD_ROOT_QUEUES
	if (unlikely(dq == &_dispatch_mgr_root_queue)) {
		pthr = _dispatch_mgr_root_queue_init();
	}
#endif
	do {
		_dispatch_retain(dq); // released in _dispatch_worker_thread
		while ((r = pthread_create(pthr, attr, _dispatch_worker_thread, dq))) {
			if (r != EAGAIN) {
				(void)dispatch_assume_zero(r);
			}
			_dispatch_temporary_resource_shortage();
		}
	} while (--remaining);
#else
	(void)floor;
#endif // DISPATCH_USE_PTHREAD_POOL
}
```

`_dispatch_root_queue_poke_slow `先判断当前队列是否有问题，接下来执行`_pthread_workqueue_addthreads `调用底层直接添加线程到工作队列；下面第一个do-while循环来判断当前队列的缓存池的大小能否继续申请线程，如果大于可申请的大小则出现容积崩溃`_dispatch_root_queue_debug("pthread pool reducing request from %d to %d",
					remaining, can_request);`，如果等于0，则报`_dispatch_root_queue_debug("pthread pool is full for root queue: "
					"%p", dq);`。如果可以开辟的话，则进入下一个do-while循环，这时我们可以发现全局并发队列`pthread_create`来创建线程，直到要创建的线程为0。

####  单例dispatch_once

进入dispatch_once源码内部`dispatch_once_f`方法内，首先对`dispatch_once_t`做标记，如果当前状态为`DLOCK_ONCE_DONE`说明有加载过下次就不再次加载；如果从来没加载过则进入`_dispatch_once_gate_tryenter`，如果当前状态是`DLOCK_ONCE_UNLOCKED`则执行`_dispatch_once_callout`内部通过`_dispatch_client_callout`来进行单例调用，`_dispatch_once_gate_broadcast`来做`DLOCK_ONCE_DONE`标记已经加载过。
```
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
	dispatch_once_gate_t l = (dispatch_once_gate_t)val;
	//DLOCK_ONCE_DONE
#if !DISPATCH_ONCE_INLINE_FASTPATH || DISPATCH_ONCE_USE_QUIESCENT_COUNTER
	uintptr_t v = os_atomic_load(&l->dgo_once, acquire);
	if (likely(v == DLOCK_ONCE_DONE)) {
		return;
	}
#if DISPATCH_ONCE_USE_QUIESCENT_COUNTER
	if (likely(DISPATCH_ONCE_IS_GEN(v))) {
		return _dispatch_once_mark_done_if_quiesced(l, v);
	}
#endif
#endif
	if (_dispatch_once_gate_tryenter(l)) {
		// 单利调用 -- v->DLOCK_ONCE_DONE
		return _dispatch_once_callout(l, ctxt, func);
	}
	return _dispatch_once_wait(l);
}
```

#### 信号量dispatch_semaphore

首先创建信号量dispatch_semaphore_create源码内部主要是初始化信号量的信息和保存信号量dsema_value。接下来进入等待信号量dispatch_wait源码内部`dispatch_semaphore_wait`，先执行`os_atomic_dec2o`对信号量-1操作后，再判断当前信号量如果大于等于0则直接返回，否则进入等待`_dispatch_semaphore_wait_slow`逻辑，其内部会一直等待直到信号量为0或者调用`semaphore_signal()`才能唤醒。
```
long
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)
{
	long value = os_atomic_dec2o(dsema, dsema_value, acquire);
	if (likely(value >= 0)) {
		return 0;
	}
	return _dispatch_semaphore_wait_slow(dsema, timeout);
}
```
再看dispatch_semaphore_signal的源码内部实现，首先等待信号量dispatch_wait正好相反，执行`os_atomic_inc2o`对信号量+1操作。
```
long
dispatch_semaphore_signal(dispatch_semaphore_t dsema)
{
	long value = os_atomic_inc2o(dsema, dsema_value, release);
	if (likely(value > 0)) {
		return 0;
	}
	if (unlikely(value == LONG_MIN)) {
		DISPATCH_CLIENT_CRASH(value,
				"Unbalanced call to dispatch_semaphore_signal()");
	}
	return _dispatch_semaphore_signal_slow(dsema);
}

long
_dispatch_semaphore_signal_slow(dispatch_semaphore_t dsema)
{
	_dispatch_sema4_create(&dsema->dsema_sema, _DSEMA4_POLICY_FIFO);
	_dispatch_sema4_signal(&dsema->dsema_sema, 1);
	return 1;
}
```
#### 调度组dispatch_group

首先进入`dispatch_group_create`源码内部，利用`_dispatch_object_alloc`来创建dispatch_group_t并初始化，最后返回。接下来看`dispatch_group_enter`，其内部先通过`os_atomic_sub_orig2o`来进行-1操作，`dispatch_group_leave`则是进行+1操作，这里可以看到如果进行`dispatch_group_enter`操作信号量不为0或者进行`dispatch_group_leave`操作后信号量等于0，则说明`dispatch_group_enter`和`dispatch_group_leave`不是匹配的，那么直接报出DISPATCH_CLIENT_CRASH信息。如果目前没问题的话那么`dispatch_group_leave`会执行`_dispatch_group_wake`，

```
DISPATCH_ALWAYS_INLINE
static inline dispatch_group_t
_dispatch_group_create_with_count(uint32_t n)
{
	dispatch_group_t dg = _dispatch_object_alloc(DISPATCH_VTABLE(group),
			sizeof(struct dispatch_group_s));
	dg->do_next = DISPATCH_OBJECT_LISTLESS;
	dg->do_targetq = _dispatch_get_default_queue(false);
	if (n) {
		os_atomic_store2o(dg, dg_bits,
				-n * DISPATCH_GROUP_VALUE_INTERVAL, relaxed);
		os_atomic_store2o(dg, do_ref_cnt, 1, relaxed); // <rdar://22318411>
	}
	return dg;
}

void
dispatch_group_leave(dispatch_group_t dg)
{
	// The value is incremented on a 64bits wide atomic so that the carry for
	// the -1 -> 0 transition increments the generation atomically.
	uint64_t new_state, old_state = os_atomic_add_orig2o(dg, dg_state,
			DISPATCH_GROUP_VALUE_INTERVAL, release);
	uint32_t old_value = (uint32_t)(old_state & DISPATCH_GROUP_VALUE_MASK);

	if (unlikely(old_value == DISPATCH_GROUP_VALUE_1)) {
		old_state += DISPATCH_GROUP_VALUE_INTERVAL;
		do {
			new_state = old_state;
			if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
				new_state &= ~DISPATCH_GROUP_HAS_WAITERS;
				new_state &= ~DISPATCH_GROUP_HAS_NOTIFS;
			} else {
				// If the group was entered again since the atomic_add above,
				// we can't clear the waiters bit anymore as we don't know for
				// which generation the waiters are for
				new_state &= ~DISPATCH_GROUP_HAS_NOTIFS;
			}
			if (old_state == new_state) break;
		} while (unlikely(!os_atomic_cmpxchgv2o(dg, dg_state,
				old_state, new_state, &old_state, relaxed)));
		return _dispatch_group_wake(dg, old_state, true);
	}

	if (unlikely(old_value == 0)) {
		DISPATCH_CLIENT_CRASH((uintptr_t)old_value,
				"Unbalanced call to dispatch_group_leave()");
	}
}

void
dispatch_group_enter(dispatch_group_t dg)
{
	// The value is decremented on a 32bits wide atomic so that the carry
	// for the 0 -> -1 transition is not propagated to the upper 32bits.
	uint32_t old_bits = os_atomic_sub_orig2o(dg, dg_bits,
			DISPATCH_GROUP_VALUE_INTERVAL, acquire);
	uint32_t old_value = old_bits & DISPATCH_GROUP_VALUE_MASK;
	if (unlikely(old_value == 0)) {
		_dispatch_retain(dg); // <rdar://problem/22318411>
	}
	if (unlikely(old_value == DISPATCH_GROUP_VALUE_MAX)) {
		DISPATCH_CLIENT_CRASH(old_bits,
				"Too many nested calls to dispatch_group_enter()");
	}
}
```
`_dispatch_group_wake`内部会通过do-while执行`_dispatch_continuation_async`来循环遍历添加到notify内的任务。这里`dispatch_group_leave`后和`_dispatch_group_notify`最后的操作一样都会调用`_dispatch_group_wake`来执行任务。
```
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_group_notify(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_continuation_t dsn)
{
	uint64_t old_state, new_state;
	dispatch_continuation_t prev;

	dsn->dc_data = dq;
	_dispatch_retain(dq);

	prev = os_mpsc_push_update_tail(os_mpsc(dg, dg_notify), dsn, do_next);
	if (os_mpsc_push_was_empty(prev)) _dispatch_retain(dg);
	os_mpsc_push_update_prev(os_mpsc(dg, dg_notify), prev, dsn, do_next);
	if (os_mpsc_push_was_empty(prev)) {
		os_atomic_rmw_loop2o(dg, dg_state, old_state, new_state, release, {
			new_state = old_state | DISPATCH_GROUP_HAS_NOTIFS;
			if ((uint32_t)old_state == 0) {
				os_atomic_rmw_loop_give_up({
					return _dispatch_group_wake(dg, new_state, false);
				});
			}
		});
	}
}

DISPATCH_NOINLINE
static void
_dispatch_group_wake(dispatch_group_t dg, uint64_t dg_state, bool needs_release)
{
	uint16_t refs = needs_release ? 1 : 0; // <rdar://problem/22318411>

	if (dg_state & DISPATCH_GROUP_HAS_NOTIFS) {
		dispatch_continuation_t dc, next_dc, tail;

		// Snapshot before anything is notified/woken <rdar://problem/8554546>
		dc = os_mpsc_capture_snapshot(os_mpsc(dg, dg_notify), &tail);
		do {
			dispatch_queue_t dsn_queue = (dispatch_queue_t)dc->dc_data;
			next_dc = os_mpsc_pop_snapshot_head(dc, tail, do_next);
			_dispatch_continuation_async(dsn_queue, dc,
					_dispatch_qos_from_pp(dc->dc_priority), dc->dc_flags);
			_dispatch_release(dsn_queue);
		} while ((dc = next_dc));

		refs++;
	}

	if (dg_state & DISPATCH_GROUP_HAS_WAITERS) {
		_dispatch_wake_by_address(&dg->dg_gen);
	}

	if (refs) _dispatch_release_n(dg, refs);
}
```
说到调度组肯定少不了`dispatch_group_async`，`dispatch_group_async`其实就是对`dispatch_group_enter`和`dispatch_group_leave`的封装。进入`dispatch_group_async`源码在初始化`_dispatch_continuation_init`保存任务后开始执行`_dispatch_continuation_group_async`操作，我们可以看到内部先进行了`dispatch_group_enter`，然后经过`_dispatch_continuation_async`、`dx_push`、`_dispatch_root_queue_poke`等操作后最终调用`_dispatch_client_callout`执行任务，当任务执行完毕后再通过mach底层来通知完成complete操作，最后执行`dispatch_group_leave`。
```
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_group_async(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_continuation_t dc, dispatch_qos_t qos)
{
	dispatch_group_enter(dg);
	dc->dc_data = dg;
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}

static inline void
_dispatch_continuation_with_group_invoke(dispatch_continuation_t dc)
{
	struct dispatch_object_s *dou = dc->dc_data;
	unsigned long type = dx_type(dou);
	if (type == DISPATCH_GROUP_TYPE) {
		_dispatch_client_callout(dc->dc_ctxt, dc->dc_func);
		_dispatch_trace_item_complete(dc);
		dispatch_group_leave((dispatch_group_t)dou);
	} else {
		DISPATCH_INTERNAL_CRASH(dx_type(dou), "Unexpected object type");
	}
}
```

该文章为记录本人的学习路程，也希望能够帮助大家，知识共享，共同成长，共同进步！！！

