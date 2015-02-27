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

### 命令行工具bin/puma

另外一种启动puma的方法



## Puma的启动流程
