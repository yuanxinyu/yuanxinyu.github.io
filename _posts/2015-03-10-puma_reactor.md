---
layout: post
title: "puma的IO处理分析"
description: ""
category : ruby
tagline: ""
tags : [puma, ruby, rack, web server]
---
{% include JB/setup %}



## puma的IO处理分析

### Reactor与Proactor两种模式
Reactor与Proactor是两种典型io事件处理模式。这两种模式都是让io数据的处理者只需要专心处理业务，而io事件的监听与通知则交给独立的第三方（一般称为事件分离者）。Reactor模式是基于同步I/O的，而Proactor模式是和异步I/O相关的。

在Reactor模式中，事件分离者等待某个事件或者可应用或个操作的状态发生（比如文件描述符可读写，或者是socket可读写），事件分离者就把这个事件传给事先注册的事件处理者（回调函数），由后者来做实际的读写操作。

而在Proactor模式中，事件处理者(或者代由事件分离者发起)直接发起一个异步读写操作(相当于请求)，而实际的工作是由操作系统来完成的。发起时，需要提供的参数包括用于存放读到数据的缓存区，读的数据大小，或者用于存放外发数据的缓存区，以及这个请求完后的回调函数等信息。事件分离者得知了这个请求，它默默等待这个请求的完成，然后转发完成事件给相应的事件处理者或者回调。

Reactor模式中，实际的io读写还是需要事件处理者完成，而Proactor模式中，事件处理者只是接收io完成的通知，实际的io已经由操作系统完成了。这里针对Reactor与Proactor的讨论比较抽象，下面的分析中看到具体的代码的时候会清楚一些。

Puma的io处理采用的是Reactor模式。


### Puma的IO总体架构

总体来看，puma的io处理有三个循环：1. Server里处理Socket连接建立的循环；2. Reactor里处理连接就绪的循环；3. 线程池里处理就绪任务的循环。

第一步：
所有的网络服务器，接入部分都是一个建立连接的循环。Puma在接受到客户端的连接请求后，就初始化一个Client对象，并将其加入线程池。

第二步：
所有建立的连接，如果还没有就绪，就加入到Reactor里等待其就绪。如果Reactor里有一个连接就绪，那么就把这个连接加入到线程池。


第三步：
线程池不断从任务队列中取出任务，执行它。如果因为io还未就绪的原因导致任务无法执行，就把这个任务（Client对象）再次加入到Reactor里。

这三个循环中，前面两个都是通过pipe接受外部控制的，第三个循环通过改变其@shutdown标志和@todo任务队列可以让其退出。

### Socket连接建立

先来看Socket连接建立的代码。前面提到，为了控制和退出这个循环，使用了pipe机制。所以整个循环其实有两块代码，处理pipe的代码和处理socket连接建立的代码。

```ruby	
  class Server
    def initialize(app, events=Events.stdio, options={})
      @app = app
      @check, @notify = Puma::Util.pipe
	  ......
	end
	  
    def handle_servers
      begin
        check = @check
        sockets = [check] + @binder.ios   ＃同时监听pipe和连接端口
        pool = @thread_pool
        queue_requests = @queue_requests

        while @status == :run
          begin
            ios = IO.select sockets			＃核心调用
            ios.first.each do |sock|
              if sock == check				＃处理pipe的事件
                break if handle_check
              else
                begin
                  if io = sock.accept_nonblock		＃处理连接的建立
                    client = Client.new io, @binder.env(sock)
                    pool << client
                    pool.wait_until_not_full unless queue_requests
                  end
                rescue SystemCallError
            end
          rescue Errno::ECONNABORTED
        end

        @events.fire :state, @status
        graceful_shutdown if @status == :stop || @status == :restart
        if queue_requests
          @reactor.clear! if @status == :restart
          @reactor.shutdown
        end
		......
      end
      @events.fire :state, :done
    end
```	

整个循环中最核心的是select(2)系统调用。要理解这块代码，先要熟悉select方法，这里先把select的说明整体摘抄下来：select monitors given arrays of IO objects, waits one or more of IO objects ready for reading, are ready for writing, and have pending exceptions respectively, and returns an array that contains arrays of those IO objects. It will return nil if optional timeout value is given and no IO object is ready in timeout seconds。[文档在这里](http://ruby-doc.org/core-2.2.1/IO.html#method-c-select)。

我们来分析这一行代码：`ios = IO.select sockets`。IO.select一共有四个参数，其中sockets是select的第一个参数，代表需要等待可读的IO对象的数组，其它的三个参数这里没用到。返回值ios是一个最多三个元素的数组（分别表示可读的／可写的／异常的），其中的每一个元素也是一个素组。所以ios.first代表可读的所有io对象。

如果可读的io对象是自己的检查pipe，那么调用handle_check处理server的停止／重启等；如果是有新的连接，那么初始化一个Client对象并加入到线程池。

```ruby	
    def handle_check
      cmd = @check.read(1)

      case cmd
      when STOP_COMMAND
        @status = :stop
        return true
      when HALT_COMMAND
        @status = :halt
        return true
      when RESTART_COMMAND
        @status = :restart
        return true
      end

      return false
    end
```	



### Reactor的循环

下面来看看Reactor的循环。Reactor里事件多路分发机制采用的也是select(2)系统调用，这是一种io多路复用的非阻塞同步io。Reactor的循环也是通过pipe来控制，所以循环代码有包含两块逻辑：pipe处理和socket就绪处理。

```ruby	
  class Reactor
    DefaultSleepFor = 5

    def initialize(server, app_pool)
      @server = server
      @app_pool = app_pool
      @ready, @trigger = Puma::Util.pipe
      @input = []
      @timeouts = []
      @sockets = [@ready]
	end
	
    def run_internal
      sockets = @sockets

      while true
        begin
          ready = IO.select sockets, nil, nil, @sleep_for 		＃核心调用
        rescue IOError => e
          if sockets.any? { |socket| socket.closed? }
            STDERR.puts "Error in select: #{e.message} (#{e.class})"
            STDERR.puts e.backtrace
            sockets = sockets.reject { |socket| socket.closed? }
            retry
          else
            raise
          end
        end

        if ready and reads = ready[0]
          reads.each do |c|
            if c == @ready 			＃处理pipe控制部分
              @mutex.synchronize do
                case @ready.read(1)
                when "*"
                  sockets += @input
                  @input.clear
                when "c"
                  sockets.delete_if do |s|
                    if s == @ready
                      false
                    else
                      s.close
                      true
                    end
                  end
                when "!"
                  return
                end
              end
            else				 		＃处理socket部分
              begin
                if c.try_to_finish
                  @app_pool << c		＃可以处理的加入线程池
                  sockets.delete c
                end

              # The client doesn't know HTTP well
              rescue HttpParserError => e
                c.write_400
                c.close

                sockets.delete c

                @events.parse_error @server, c.env, e
              rescue StandardError => e
                c.write_500
                c.close

                sockets.delete c
              end
            end
          end
        end

        unless @timeouts.empty?
          @mutex.synchronize do
            now = Time.now

            while @timeouts.first.timeout_at < now
              c = @timeouts.shift
              c.write_408 if c.in_data_phase
              c.close
              sockets.delete c

              break if @timeouts.empty?
            end

            calculate_sleep
          end
        end
      end
    end
	
```	

代码中的pipe控制逻辑有三种情况：“*”,"c"和"!"。“*”代表增加一个待监控的客户端socket连接，"c"代表清空reactor中的sockets连接, "!"代表关闭reactor。下面是给reactor增加客户端连接的方法，可以看到主要就是把连接c加入到@input变量中，然后向pipe写入"*"。这样在主循环中收到pipe的写入字符就知道要增加一个客户端连接了。


```ruby	
    def add(c)
      @mutex.synchronize do
        @input << c
        @trigger << "*"

        if c.timeout_at
          @timeouts << c
          @timeouts.sort! { |a,b| a.timeout_at <=> b.timeout_at }
          calculate_sleep
        end
      end
    end
```	


循环中的另外一半处理socket连接。Socket连接就绪时，通过运行try_to_finish判断http请求是否已经可以处理，如果可以的话，把socket连接加入到线程池。其它的大部分代码都是处理异常的，http协议错误返回400，连接超时错误返回408，其它的错误都返回500。


```ruby	
  class Client
    def try_to_finish
      return read_body unless @read_header
      data = @io.read_nonblock(CHUNK_SIZE)  ＃reactor模式下，还是需要事件处理方自己读io
      @buffer << data
      @parsed_bytes = @parser.execute(@env, @buffer, @parsed_bytes)
      if @parser.finished?
        return setup_body
      end
      false
    end
```	
从上面的代码可以看出，reactor模式下还是需要事件处理方自己读io。这是reactor模式与proactor的区别。当采用proactor模式，io是操作系统完成的，事件处理方只需要处理io完成后的部分。代码中处理http协议解析的部分`parser.execute`其实是C语言实现的，这部分后面再单独分析。先看看parser完成后设置body的部分。

```ruby	
  class Client
    EmptyBody = NullIO.new

    def setup_body
      @in_data_phase = true
      body = @parser.body
      cl = @env[CONTENT_LENGTH]

      unless cl
        @buffer = body.empty? ? nil : body
        @body = EmptyBody
        @requests_served += 1
        @ready = true
        return true
      end

      remain = cl.to_i - body.bytesize

      if remain <= 0
        @body = StringIO.new(body)
        @buffer = nil
        @requests_served += 1
        @ready = true
        return true
      end

      if remain > MAX_BODY      ＃1024 * (80 + 32)
        @body = Tempfile.new(Const::PUMA_TMP_BASE)
        @body.binmode
      else
        # The body[0,0] trick is to get an empty string in the same
        # encoding as body.
        @body = StringIO.new body[0,0]
      end

      @body.write body
      @body_remain = remain
      @read_header = false
      return false
    end
```	
从代码中可以看出，body存在三种可能，如果http请求没有body部分，那么就是EmptyBody；如果body大于112Kb，那么body的内容独立保存为一个Tempfile；其它情况下，body是一个StringIO对象。


### 线程池

最后来看看线程池的处理循环。

```ruby	
  class ThreadPool
    def initialize(min, max, *extra, &block)
      @not_empty = ConditionVariable.new
      @not_full = ConditionVariable.new
      @mutex = Mutex.new
      @todo = []		＃待处理任务
      @workers = []		＃所有的工作线程
      @mutex.synchronize do
        @min.times { spawn_thread }
      end
	  ......
    end

    def spawn_thread
      @spawned += 1

      th = Thread.new do
        todo  = @todo
        extra = @extra.map { |i| i.new }

        while true
          work = nil
          continue = true
          mutex.synchronize do
            while todo.empty?
              if @trim_requested > 0
                @trim_requested -= 1
                continue = false
                break
              end

              if @shutdown
                continue = false
                break
              end

              @waiting += 1
              not_full.signal
              not_empty.wait mutex
              @waiting -= 1
            end

            work = todo.shift if continue  ＃取出任务
          end
          break unless continue
          block.call(work, *extra)     # 实际执行任务
        end

        mutex.synchronize do
          @spawned -= 1
          @workers.delete th
        end
      end

      @workers << th
      th
    end
```	
线程池里有多个工作线程，保存在@workers中。每一个工作线程有一个while循环，不断从任务队列@todo中取可以执行的任务。线程之间用mutex进行同步。每一个任务的实际执行代码是`block.call(work, *extra)`，其中的block是线程池在初始化的时候传递进来的，而work是一个Client对象。下面是block的代码：

```ruby	
  class Server
    def run
	  ......
      @thread_pool = ThreadPool.new(@min_threads,
                                    @max_threads,
                                    IOBuffer) do |client, buffer|
        process_now = false
        begin
          if queue_requests
            process_now = client.eagerly_finish
          else
            client.finish
            process_now = true
          end
        rescue HttpParserError => e
          client.write_400
          client.close

          @events.parse_error self, client.env, e
        rescue ConnectionError
          client.close
        else
          if process_now
            process_client client, buffer
          else
            client.set_timeout @first_data_timeout
            @reactor.add client
          end
        end
      end
```	
可见，线程池执行的block代码块是Server启动的时候在初始化ThreadPool时设置的。实际的http请求处理在process_client方法中处理，这块代码下一篇再分析。对于现在还不能处理的客户端，把它加入reactor中。

下面是把任务加入到线程池的代码：

```ruby	
  class ThreadPool
    def <<(work)
      @mutex.synchronize do
        if @shutdown
          raise "Unable to add work while shutting down"
        end

        @todo << work

        if @waiting < @todo.size and @spawned < @max
          spawn_thread
        end

        @not_empty.signal
      end
    end
```	
当线程池初始化的时候，先执行@min次`spawn_thread`方法，然后在加任务的时候，如果线程数还没有达到@max，动态增加线程。

最后，看一下如何停止线程池：

```ruby	
    def shutdown
      @mutex.synchronize do
        @shutdown = true
        @not_empty.broadcast
        @not_full.broadcast

        @auto_trim.stop if @auto_trim
      end

      # Use this instead of #each so that we don't stop in the middle
      # of each and see a mutated object mid #each
      if !@workers.empty?
          @workers.first.join until @workers.empty?
      end

      @spawned = 0
      @workers = []
    end
```	

### 输出
Puma对输入的socket连接的处理很复杂，而对输出的处理则简单很多，代码如下：

```ruby	
    def fast_write(io, str)
      n = 0
      while true
        begin
          n = io.syswrite str
        rescue Errno::EAGAIN, Errno::EWOULDBLOCK
          if !IO.select(nil, [io], nil, WRITE_TIMEOUT)
            raise ConnectionError, "Socket timeout writing data"
          end

          retry
        rescue  Errno::EPIPE, SystemCallError, IOError
          raise ConnectionError, "Socket timeout writing data"
        end

        return if n == str.bytesize
        str = str.byteslice(n..-1)
      end
    end
```	
所有的输出最后都是调用syswrite实现的。Syswrite是一种底层的写方法，它在写数据时不使用ruby层的缓冲。Syswrite不能和普通的write混合使用。

