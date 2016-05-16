---
layout: post
title: ceph-消息处理模块 
tags: ceph

---


# Ceph解析-消息处理模块

## 数据结构
[]()

Ceph的消息处理是采用发布订阅的设计模式,其中Messenger作为消息的发布者，
Dispatcher和其子类为消息的订阅者。

### Messenger
Messenger 为消息的发布者，根据处理的网络包的处理不同其接口为不同的子类
实现。Messenger的功能就是作为消息的发布者，从Pipe读取到消息后，将信息
转发给Dispatcher和其子类处理。

### Simple Messenger
Simple Messenger为Messenger的子类，其接口的实现。Simple Messenger处理
网络包的方式为对于为每个请求新建线程处理，为经典的生产者消费者问题。

### Dispatcher
Dispatcher为订阅者的基类，不同的子类实现功能不同。Dispatcher在初始化时
注册到发布者Messenger上，接受到Messenger的消息后再进行处理。

### Pipe
Pipe主要是消息的读取和发送，主要有两个组件Reader和Writer，分别用来处理
socket上的读取和发送，不仅是消息还有ACK和其他。Reader和Writer都是Thread
的子类，也就是说每次处理消息的的时候也会新建者两个线程。消息经过Reader
处理后，Messenger会通知已注册的Dispatcher处理，处理完后将回复的消息放到
Messenger::Pipe::out_q队列中，然后Writer再处理发送。

### Accepter
Accepter监听请求，监听到请求后，Messenger新建Pipe处理。

### DispatchQueue
DispatchQueue包含了需要处理的消息，通过消息的优先级组织好，再通过DisapatchThread分发给Dispatcher处理。DispatchThread启动后，进入
等待状态，当有消息被放入队列时被激活。

## 消息处理流程
[]()

### 初始化
Messenger 初始化时，由于Messenger是抽象类，不能直接实例化。通过调用create
方法实例化一个Simple Messenger.
```
// 调用create方法实例化一个Simple Messengern
oMessenger *msgr = Messenger::create(g_ceph_context, g_conf->ms_type,  
				      entity_name_t::MON(rank), "mon", 0);
// Messenger: bind() 绑定ip地址
//     Simple Messenger: bind()
//           Accepter: bind()
//           create socket and start listening
err = msgr->bind(ipaddr);
```

Dispatcher的初始化，以Mon为例来解析。Mon初始化时，向Messenger注册, 然后
Messenger准备好。 在Messenger:ready()中DispatchQueue 启动线程，开始接收
需要缓存的消息。Accepter 启动线程，开始监听新的请求。
```
// start monitor
mon = new Monitor(g_ceph_context, g_conf->name.get_id(), store,
		    msgr, &monmap);
// Monitor: init() 
//   Messenger: add_dispatcher_tail()
//   Messenger: ready()
//      DispatchQueue: start() create Dispathce Thread
//      Accepter: start() create thread to handle entry()
//           Accepter: entry()  socket accept message and messenger
//                              add a accept pipe to handle it
//                 Meessenger: add_accept_pipe(sd)  start reader thread
//                     Pipe: start_reader()
//                         Pipe: reader()
mon->init();
```
### 消息处理-读
Dispathcer(Mon)和Messenger初始化后， Messenger在ready()启动DispatchQueue
线程和Accepter线程。DispatchQueue: in_q 开始缓存需要处理的消息，然后交给
Dispatcher处理，Dispatcher处理后再将回复的消息放到 out_q。 Accepter启动线
程，开始在entry()接收请求和接收到消息后Messenger创建一个Pipe和启动Pipe:
start_reader()读取消息，将消息放入DispatchQueue: in_q。

```
void SimpleMessenger::ready()
{
// 启动DispatchQueue线程
  dispatch_queue.start();
// 启动Accepter线程
  lock.Lock();
  if (did_bind)
    accepter.start();
  lock.Unlock();
}
```
Accepter启动线程，线程都是由Thread为基类，Thread:create()调用phtread_
create新建线程，start_routine函数为entry().
```
int Accepter::start()
{
// start thread, handle entry()
  create();
  return 0;
}
```

Accepter::entry() 为Accepter线程的start_rountine，开始接收请求并交给
Simple Messenger处理。
```
void *Accepter::entry()
{
  struct pollfd pfd;
  pfd.fd = listen_sd;
  pfd.events = POLLIN | POLLERR | POLLNVAL | POLLHUP;
  while (!done) {
    int r = poll(&pfd, 1, -1);
    if (done) break;
    // accept 接收连接请求
    entity_addr_t addr;
    socklen_t slen = sizeof(addr.ss_addr());
    int sd = ::accept(listen_sd, (sockaddr*)&addr.ss_addr(), &slen);
    if (sd >= 0) {
	    // Simple Messenger开始处理这个连接
		msgr->add_accept_pipe(sd);
    } 
  }
  return 0;
}
```

Messenger:add_accept_pipe(sd) 新建一个Pipe然后调用start_reader()处理这
个连接，start_reader启动Reader线程调用Pipe:reader()读取信息。

```
Pipe *SimpleMessenger::add_accept_pipe(int sd)
{
  lock.Lock();
  Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL);
  p->sd = sd;
  p->pipe_lock.Lock();
  // 调用start_reader开始处理
  p->start_reader();
  p->pipe_lock.Unlock();
  pipes.insert(p);
  accepting_pipes.insert(p);
  lock.Unlock();
  return p;
}
```
```
void Pipe::reader()
{
  // Pipe:accept()，调用start_writer()启动Writer线程，writer
  // 线程进入cond.Wait()等待被激活。
  if (state == STATE_ACCEPTING) {
    accept();
  }
  // loop.
  while (state != STATE_CLOSED && state != STATE_CONNECTING) {
    char tag = -1;
	// 读取信息标签，根据消息类型不同进行不同处理
    if (tcp_read((char*)&tag, 1) < 0) {
      pipe_lock.Lock();
      continue;
    }

    if (tag == CEPH_MSGR_TAG_KEEPALIVE) {2
    }
    if (tag == CEPH_MSGR_TAG_KEEPALIVE2) {
    }
    if (tag == CEPH_MSGR_TAG_KEEPALIVE2_ACK) {
    }
    if (tag == CEPH_MSGR_TAG_ACK) {
    }
    else if (tag == CEPH_MSGR_TAG_MSG) {
	  // 读取消息
      Message *m = 0;
      int r = read_message(&m, auth_handler.get());

	  // 激活writer线程，回复ACK
      cond.Signal();  // wake up writer, to ack this
      
      in_q->fast_preprocess(m);

	  // 根据消息的不同进行不同的处理
	  // 延时处理
	  // 快速处理
	  // 放入队列根据优先级
      if (delay_thread) {
        delay_thread->queue(release, m);
      } else {
        if (in_q->can_fast_dispatch(m)) {
		  
          in_q->fast_dispatch(m);
		  
		  if (state == STATE_CLOSED ||notify_on_dispatch_done) {
	  		  // there might be somebody waiting
			  notify_on_dispatch_done = false;
			  cond.Signal();
		  }
        } else {
		  // 放入dispatchQueue队列里，交给dispatcher处理
          in_q->enqueue(m, m->get_priority(), conn_id);
        }
      }
    }
}
```
### 消息处理-写
Writer线程启动后,start_rontine为Pipe::writer(), writer进入等待
状态，等待被激活，激活后从Map<int, list<Message*> >::out_q中读取消息，然后发送消息。

```
void Pipe::writer()
{
  pipe_lock.Lock();
  
  while (state != STATE_CLOSED) {// && state != STATE_WAIT) {
    if (state != STATE_CONNECTING && state != STATE_WAIT &&
		state != STATE_STANDBY && (is_queued() || in_seq >
		in_seq_acked)) {
        // handle keekalive etc
		....
		
		// grab outgoing message from out_q for sending
		Message *m = _get_next_outgoing();
		
		if (m) {
		
			// encode and copy out of *m
			m->encode(features, msgr->crcflags);
			
			// prepare everything, get message header
			const ceph_msg_header& header = m->get_header();
			const ceph_msg_footer& footer = m->get_footer();
			
			bufferlist blist = m->get_payload();
			blist.append(m->get_middle());
			blist.append(m->get_data());

	        // send message
			// write_message()->Pipe::do_sendmsg()->::sendmsg()
	        int rc = write_message(header, footer, blist);
			m->put();
		}
	}
	// wait for wake up
	cond.Wait(pipe_lock);
  }
}
```
## 消息处理流程

Reader线程将读取的消息放入DispatchQueue::in_q后, DispathchQueue
将消息放入优先级队列mqueue中，然后激活DispatchThread线程，在线程
的start_routine函数DispatcheQueue::entry()中，调用Messenger::
ms_deliver_dispatch()函数将消息分发给所有已注册的Dispatcher，这
里以Monitor为例，Monitor在_ms_dispatch()进一步在dispatch_op()中
根据消息的不同调用不同的handle函数处理, 如PING操作，调用handle_
ping(), 在handle_ping()处理完后，将回复消息通过Messenger::send_
reply()再调用Messenger::_send_message()通过_submit_message()函数
新建一个Pipe，利用Pipe::_send()将回复消息放入Map<int,
list<Message*> >::out_q, 最后激活Writer线程。

```
Pipe:: entry() { in_q.enqueue(...) }
--> DispathcQueue:: enqueue() {mqueue.enqueue(..); cond.Wait()}
---> DispatchQueue:: entry() {Messenger::ms_deliver_dispatch()}
----> Messenger:: ms_deliver_dispatch() {Dispatcher::
	                                     ms_dispatch() }
-----> Monitor::ms_dispatch() { _ms_dispatch()}
------> Monitor::_ms_dispatch() { dispatch_op()}
-------> Monitor:: dispatch_op() { handle_ping()}
--------> Monitor::handle_ping() {messenger->send_reply()}
---------> SimpleMessenger::send_reply(){ _send_message()}
----------> SimpleMessenger::_send_message() {submit_message()}
-----------> SimpleMessenger::submit_message() {Pipe::send()}
------------> Pipe::_send(){out_q[prt]push_back(); cond.Wait()}
```

```
void DispatchQueue::entry()
{
  while (true) {
    while (!mqueue.empty()) {
      QueueItem qitem = mqueue.dequeue();
      if (qitem.is_code()) {
	  ....
      } else {
	      // 取出消息
		  Message *m = qitem.get_message();
		  if (stop) {
			  m->put();
		  } else {
		     uint64_t msize = pre_dispatch(m);
			 // 调用Messenger进行消息的分发
			 msgr->ms_deliver_dispatch(m);
			 post_dispatch(m, msize);
		 }
      }
    }
    if (stop)
      break;
	  
    // wait for something to be put on queue
	// 在DispatchQueue::enqueue()被激活
    cond.Wait(lock);
  }
}
```

## 参考
[Ceph深入解析:消息架构模块](http://mathslinux.org/?p=664)
[Ceph源码解析:网络模块](http://hustcat.github.io/ceph-internal-network/)




