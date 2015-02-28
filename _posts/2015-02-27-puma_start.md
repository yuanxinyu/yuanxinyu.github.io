---
layout: post
title: "puma的启动流程"
description: ""
category : ruby
tagline: ""
tags : [puma, ruby, rack, web server]
---
{% include JB/setup %}



## Puma的启动方式

puma有两种启动方式: 一种是通过标准的rack接口启动, 一种是通过puma提供的命令行工具启动。

### rack接口

首先，Puma会把自己设置为缺省的rack handler，见文件lib/puma/rack_default.rb

```ruby
	module Rack::Handler
	  def self.default(options = {})
	    Rack::Handler::Puma
	  end
	end
```

然后，在Rack::Handler::Puma的run方法中启动puma，见文件lib/rack/handler/puma.rb

```ruby
        def self.run(app, options = {})
          ......
          server   = ::Puma::Server.new(app)
          min, max = options[:Threads].split(':', 2)
  
          puts "Puma #{::Puma::Const::PUMA_VERSION} starting..."
          puts "* Listening on tcp://#{options[:Host]}:#{options[:Port]}"
  
          server.add_tcp_listener options[:Host], options[:Port]
          server.min_threads = min
          server.max_threads = max
          yield server if block_given?
  
          begin
            server.run.join
          rescue Interrupt
            puts "* Gracefully stopping, waiting for requests to finish"
            server.stop(true)
            puts "* Goodbye!"
          end
        end
```  

  
run方法中的参数app是一个rack应用，options只支持主机名／端口号／线程数／日志等有限的几个参数。从代码中可见，rack接口启动puma只监听tcp端口，且使用单进程模式，不支持集群。所以通过rack接口启动puma不能完全利用puam的高级功能。

通过下面的命令都会以rack接口启动puma：

	rackup -s Puma
	rails s Puma

### 命令行工具bin/puma

另外一种启动puma的方法是通过命令行的bin/puma命令, 这是一个ruby写的可执行脚本，代码如下：

```ruby
	require 'puma/cli'
	cli = Puma::CLI.new ARGV
	cli.run
```

可见，实际处理命令行参数和启动puma的是Puma::CLI类。命令行启动的时候，可以传递一些参数给puma。参数有两种格式：短的和长的，比如-p 和 --port都是设置监听端口。你可以在命令行把所有的puma参数都设置好，也可以把设置参数写在一个文件里，然后通过-C来引用。

	$ puma -b tcp://127.0.0.1:9292 -t 8:32 -w 3
	$ puma -C /path/to/config


CLI启动puma的具体流程见下一节。


## Puma的启动流程
这里主要讲解命令行启动puma的流程。启动过程分两步，首先是初始化Puma::CLI类，然后执行cli.run方法。

### Puma的CLI初始化

CLI初始化的代码如下：

```ruby
    def initialize(argv, events=Events.stdio)
	  ......
      @events = events
      setup_options
      generate_restart_data
      @binder = Binder.new(@events)
      @binder.import_from_env
    end
```
主要步骤包括：初始化事件的标准输出与错误输出、设置命令行参数的缺省值、设置解析命令行参数的代码、设置重启puma的命令行、初始化Binder类。

setup_options方法首先设置命令行参数的缺省值，比如缺省的最小和最大线程数是0和16。然后初始化@parser对象，实现命令行参数的解析。解析代码也很简单，每找到一个命令行参数，就把它加入@options数组。

```ruby
    def setup_options
      @options = {
        :min_threads => 0,
        :max_threads => 16,
		......
      }

      @parser = OptionParser.new do |o|
        o.on "-b", "--bind URI", "URI to bind to (tcp://, unix://, ssl://)" do |arg|
          @options[:binds] << arg
        end
		......
	  end
	end
```

然后是获取重启puma的命令行，并保存到@restart_argv中。

```ruby
    def generate_restart_data
      @restart_dir ||= Dir.pwd
      @original_argv = ARGV.dup
      if File.exist?($0)
        arg0 = [Gem.ruby, $0]
      else
        arg0 = [Gem.ruby, "-S", $0]
      end
      # Detect and reinject -Ilib from the command line
      lib = File.expand_path "lib"
      arg0[1,0] = ["-I", lib] if $:[0] == lib
	  @restart_argv = arg0 + ARGV
    end
```
由于generate_restart_data方法比较难懂，这里演示一下阅读puma源代码时的调试过程。首先下载puma的源代码并编译。

	ylt~/tmp/$ git clone https://github.com/puma/puma.git
	ylt~/tmp/$ cd puma
	ylt~/tmp/puma$ rake

然后在文件lib/puma/cli.rb的374行设置断点，我使用的是byebug。接着使用下面的命令启动puma：

```shell
	ylt~/tmp/puma$ ruby -I lib/ bin/puma
	[371, 380] in /Users/ylt/tmp/puma/lib/puma/cli.rb
	   371:         else
	   372:           @restart_argv = arg0 + ARGV
	   373:         end
	   374:         byebug
	   375:       end
	=> 376:     end
	   377: 
	   378:     def restart_args
	   379:       if cmd = @options[:restart_cmd]
	   380:         cmd.split(' ') + @original_argv
	(byebug) @restart_argv
	["/Users/ylt/.rvm/rubies/ruby-2.1.2/bin/ruby", "-I", "/Users/ylt/tmp/puma/lib", "bin/puma"]
	(byebug) $0
	"bin/puma"
```

代码中有不懂的地方，直接在调试环境中执行代码，比如查看@restart_argv和$0到底是什么值。
如果想知道restart_argv到底有什么用，用grep搜索代码目录，看看在哪个地方使用了这个变量。最终会发现只有一个地方使用了它，简化后的代码是：

```ruby
    def restart!
        argv = restart_args
        Dir.chdir @restart_dir
        argv += [redirects] unless RUBY_VERSION < '1.9'
        Kernel.exec(*argv)
	end
```
也就是通过Kernel.exec来执行命令行来重启。


###Puma的运行流程
完成Puma::CLI的初始化以后，调用其run方法将puma实际运行起来，这个run方法是puma执行的主函数体。

```ruby
    def run
      parse_options
      set_rack_environment
      if clustered?
        @events.formatter = Events::PidFormatter.new
        @options[:logger] = @events
        @runner = Cluster.new(self)
      else
        @runner = Single.new(self)
      end
      setup_signals
      set_process_title
      @status = :run
      @runner.run
	  # 这里等待runner运行结束，一般是收到了KILL类信号
      case @status
      when :halt
        log "* Stopping immediately!"
      when :run, :stop
        graceful_stop
      when :restart
        log "* Restarting..."
        @runner.before_restart
        restart!
      when :exit
        # nothing
      end
    end
```

Puma运行的第一步是解析命令行参数，然后设置rack环境。下一步判断是否运行为集群模式，如果是集群模式，初始化Cluster类；如果是单进程模式，初始化Single类。然后是设置操作系统信号的处理函数、设置puma进程标题。紧接着是设置puma主进程的状态为:run，并执行runner的run方法。run方法进行几层的封装，最终会执行到Server类的run方法，其中的关键循环是：

```ruby
        while @status == :run
          begin
            ios = IO.select sockets
			...
		end
```


此时puma服务器就可以接收web客户端的请求了，同时等待操作系统的退出信号。当收到操作系统的退出信号后，代码执行到case @status处，这里判断信号类型来决定是停止还是重启puma。


