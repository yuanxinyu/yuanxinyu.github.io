---
layout: post
title: "puma的单进程模式"
description: ""
category : ruby
tagline: ""
tags : [puma, ruby, rack, web server]
---
{% include JB/setup %}



## Puma的单进程运行模式

### Puma的类结构

在puma的启动过程代码中可以看到，负责单进程运行模式的类是Single。在分析单进程运行模式之前，先总体看一下puma的单进程／集群架构。其类图如下所示，可以看出很像策略模式。

![](/dot/4.png)

最右边是puma的启动入口类PUAM::CLI，它包含一个成员变量@runner，属于Runner类。Runner类有两个子类Single和Cluster，分别负责单进程和集群运行模式。Single和Cluster都提供一个run接口，CLI的run方法会调用对应类的run方法。可以看出，这类似GOF的策略模式。但是ruby是动态语言，puma的作者其实应该不是按照策略模式来实现这套框架的。

下面是puma早期的代码：

```ruby
      clustered = @options[:workers] > 0
      if clustered
        run_cluster
      else
        run_single
      end
```
可见在puma早期，单进程执行run_single方法，集群执行run_cluster方法，没有Runner/Single/Cluster的类继承体系，CLI里直接写单进程和集群的运行模式代码。

所以，现在的这个架构是逐步重构出来的。Single和Cluster有很多雷同的代码，于是增加一个Runner类，把共同的代码提取到基类中，形成了现在的架构。我自己本人也是这样的编程习惯，通过不断的重构，清除重复代码，厘清代码的结构。

Runner中有四个成员变量，分别是cli／options／app／control，各代表命令行启动类，命令行参数类、配置的rack app，运行时控制puma的PUMA::ControlCLI类。这四个类对单进程和多进程来说都是一样的。然后Single包含一个PUAM::Server类，而Cluster则包含一个PUMA::Worker的数组，这是单进程与集群模式真正有区别的地方。


### 单进程的执行流程
单进程的执行流程由Single类的run方法实现，它首先判断是否要以daemon的方法运行puma，然后加载配置文件并确定监听哪些网络接口，接着把puma的进程pid以及状态信息写入状态文件中，然后启动运行时控制服务器的网络接口，然后通知启动监听者puma启动了，最后是运行Server类的run方法。

```ruby
	  class Single < Runner
	      def run
             if daemon?
               log "* Daemonizing..."
               Process.daemon(true)
               redirect_io
             end
	        load_and_bind
	        @cli.write_state
	        start_control
	        @server = server = start_server
	        @cli.events.fire_on_booted!
	        begin
	          server.run.join
	        rescue Interrupt
	          # Swallow it
	        end
	  end		
```

#### 绑定网络端口
绑定网络端口的实现代码如下：

```ruby
class Runner
    def load_and_bind
      ……
      begin
        @app = @cli.config.app
      rescue Exception => e
        log "! Unable to load application: #{e.class}: #{e.message}"
        raise e
      end
      @cli.binder.parse @options[:binds], self  ＃实际绑定网络接口发生在这里
    end
end
```

首先加载Rack的@app，然后调用Binder类的parse方法实现网络绑定端口的解析和绑定。我感觉这个parse方法名字取的不好，叫parse_and_bind会更合适一些。另外一个违反直觉的是启动TCPServer的代码在Binder类中，而不在Server类中。下面看看parse方法的实现：

```ruby
class Binder
    def parse(binds, logger)
      binds.each do |str|
        uri = URI.parse str
        case uri.scheme
        when "tcp"
          params = Rack::Utils.parse_query uri.query
          opt = params.key?('low_latency')
          bak = params.fetch('backlog', 1024).to_i
          logger.log "* Listening on #{str}"
          io = add_tcp_listener uri.host, uri.port, opt, bak
          @listeners << [str, io]
        when "unix"
          path = "#{uri.host}#{uri.path}".gsub("%20", " ")
          logger.log "* Listening on #{str}"
          umask = nil
          mode = nil
          if uri.query
            params = Rack::Utils.parse_query uri.query
            if u = params['umask']
              # Use Integer() to respect the 0 prefix as octal
              umask = Integer(u)
            end
            if u = params['mode']
              mode = Integer('0'+u)
            end
          end
          io = add_unix_listener path, umask, mode
          @listeners << [str, io]
        when "ssl"
		  ......
		 end
    end
end
```

这里的核心代码是add_tcp_listener和add_unix_listener，分别启动TCPServer和UNIXServer并监听对应的网络端口。Puma可以一次监听多个网络端口，其中unix快于tcp，tcp快于ssl。由于实际应用中，ssl都是前端接入的web服务器做ssl offloading，所以这里的ssl相关代码就不分析了。

```ruby
    def add_tcp_listener(host, port, optimize_for_latency=true, backlog=1024)
      host = host[1..-2] if host[0..0] == '['
      s = TCPServer.new(host, port)
      if optimize_for_latency
        s.setsockopt(Socket::IPPROTO_TCP, Socket::TCP_NODELAY, 1)
      end
      s.setsockopt(Socket::SOL_SOCKET,Socket::SO_REUSEADDR, true)
      s.listen backlog
      @ios << s
      s
    end
	
    def add_unix_listener(path, umask=nil, mode=nil)
      @unix_paths << path
	  ......
      s = UNIXServer.new(path)
      @ios << s
      env = @proto_env.dup
      env[REMOTE_ADDR] = "127.0.0.1"
      @envs[s] = env
      s
    end
```

#### 写状态文件

当网络端口绑定以后，下一步是写入pid文件和状态文件，这两个文件的地址都是在config文件中通过pidfile／state_path命令配置的。

```ruby
    def write_state
      write_pid
      require 'yaml'
      if path = @options[:state]
        state = { "pid" => Process.pid }
        cfg = @config.dup
        state["config"] = cfg
        File.open(path, "w") do |f|
          f.write state.to_yaml
        end
      end
    end

    def write_pid
      if path = @options[:pidfile]
        File.open(path, "w") do |f|
          f.puts Process.pid
        end
        cur = Process.pid
        at_exit do
          if cur == Process.pid
            delete_pidfile
          end
        end
      end
    end
```	

写状态文件的代码很简单，基本就是把Configuration的对象实例序列化为yaml格式并写入文件中。写pid文件也很简单，只是ruby进程退出的时候要把老的pid文件删除。

下一步是启动puma的control server，这是通过在config文件中配置control_url实现的。这块代码和主流程没有太大关系，先放放。

#### 启动Server
```ruby
  class Runner
    def start_server
      min_t = @options[:min_threads]
      max_t = @options[:max_threads]

      server = Puma::Server.new app, @cli.events, @options
      server.min_threads = min_t
      server.max_threads = max_t
      server.inherit_binder @cli.binder

      if @options[:mode] == :tcp
        server.tcp_mode!
      end
      unless development?
        server.leak_stack_on_error = false
      end
      server
    end
```
启动Server的方法start_server其实只是初始化了一个Server类的实例，赋值一些变量。

```ruby
  class Server
    def initialize(app, events=Events.stdio, options={})
      @app = app
      @events = events
      @check, @notify = Puma::Util.pipe
      @status = :stop
      @min_threads = 0
      @max_threads = 16
      @auto_trim_time = 1
      @thread = nil
      @thread_pool = nil
      @persistent_timeout = PERSISTENT_TIMEOUT
      @binder = Binder.new(events)
      @own_binder = true
      @first_data_timeout = FIRST_DATA_TIMEOUT
      @leak_stack_on_error = true
      @options = options
      @queue_requests = options[:queue_requests].nil? ? true : options[:queue_requests]
      ENV['RACK_ENV'] ||= "development"
      @mode = :http
    end
```
一个Server类用来服务一个rack app。

#### boot的事件通知
当Server初始化完成以后，通知事件监听者puma启动了。

```ruby
  class Events
    def fire_on_booted!
      @on_booted.each { |b| b.call }
    end
```

	
#### Server的run方法
单进程模式的最后一步是调用Server的run方法。如果background设置为true，那么会在一个新线程中运行实际的handle_servers代码，否则会同步执行。
	
```ruby	
  class Server	
    def run(background=true)
      BasicSocket.do_not_reverse_lookup = true
      @events.fire :state, :booting
      @status = :run
      queue_requests = @queue_requests
      @thread_pool = ThreadPool.new(@min_threads,
                                    @max_threads,
                                    IOBuffer) do |client, buffer|
        ＃当有数据来的时候的回调block，后面详细分析io代码时分析此处省略的代码。
      end
      if queue_requests
        @reactor = Reactor.new self, @thread_pool
        @reactor.run_in_thread
      end
      @events.fire :state, :running
      if background
        @thread = Thread.new { handle_servers }
        return @thread
      else
        handle_servers
      end
    end
```	
这一章里就不讨论io的具体实现（io也是最复杂的一块），先看总体流程。

```ruby	
    def handle_servers
      begin
        check = @check
        sockets = [check] + @binder.ios
        pool = @thread_pool
        queue_requests = @queue_requests

        while @status == :run
          begin
            ios = IO.select sockets
            ＃网络数据处理部分，后面详细分析io代码时分析此处省略的代码。
          rescue Errno::ECONNABORTED
            # client closed the socket even before accept
            client.close rescue nil
          rescue Object => e
            @events.unknown_error self, e, "Listen loop"
          end
        end

        @events.fire :state, @status
        graceful_shutdown if @status == :stop || @status == :restart
        if queue_requests
          @reactor.clear! if @status == :restart
          @reactor.shutdown
        end
        @check.close
        @notify.close
        if @status != :restart and @own_binder
          @binder.close
        end
      end
      @events.fire :state, :done
    end
```
Puma服务器通过@status变量控制其运行。当@status为:run的时候，服务器处于运行状态，当外部操作导致@status变化时，puma进入关闭／重启流程。
