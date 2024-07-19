# 令牌桶限流测试流程

1. 拉取我个人仓库的代码，并切换到 **branch_token_bucket_test** 分支

   ```shell
   git clone https://github.com/leolin49/trpc-cpp.git && git checkout branch_token_bucket_test
   ```

2. 运行项目根目录中的 **token_bucket_test.sh** 脚本（编译项目和examples/helloworld，并运行trpc服务器）

   ```
   . token_bucket_test.sh
   ```

   脚本中启动trpc服务器的配置文件为 **trpc/overload_control/token_bucket_limiter/token_bucket_overload_ctrl.yaml**

   其中关于令牌桶限流插件的配置如下（**令牌桶容量为50，令牌产生的速率为5个/秒，初始令牌数量为0**）：

   ```yaml
   #Server configuration
   server:
     service: #Business service, can have multiple.
       - name: trpc.test.helloworld.Greeter #Service name, needs to be filled in according to the format, 
         filter:
           - token_bucket_limiter
   
   #Plugin configuration.
   plugins:
     overload_control:
       token_bucket_limiter:
         capacity: 50 # Maximum token bucket count capacity. It is configured small for unit testing purposes, but users can configure it to be larger.
         initial_token: 0 # Initial token count.
         rate: 5 # Token production rate(count/second).
         is_report: true # Whether to report
   ```

3. 打开另外一个终端，运行客户端测试程序。

   ```sh
   ./bazel-bin/examples/helloworld/test/fiber_client --client_config=./examples/helloworld/test/conf/trpc_cpp_fiber.yaml
   ```

   客户端程序中新建了40个线程并同时向**trpc server**发送**SayHello**请求

   ```c++
   std::vector<std::thread> threads;
   for (int i = 1; i <= 40; i++) {   
     auto proxy = ::trpc::GetTrpcClient()->GetProxy<::trpc::test::helloworld::GreeterServiceProxy>(FLAGS_service_name);
     threads.emplace_back([i,proxy](){
         DoRpcCall(proxy, i); 
     }); 
   }
   for (auto& t : threads) {
     if (t.joinable()) {
         t.join();
     }   
   }
   ```

4. 多执行几次步骤3中的指令，即可触发令牌桶限流过载保护的效果；客户端程序输入如下（可以看到部分请求因为拿不到令牌而被trpc server拒绝）

   ```
   [root@localhost trpc-cpp]# ./bazel-bin/examples/helloworld/test/fiber_client --client_config=./examples/helloworld/test/conf/trpc_cpp_fiber.yaml
   FLAGS_service_name:trpc.test.helloworld.Greeter
   FLAGS_client_config:./examples/helloworld/test/conf/trpc_cpp_fiber.yaml
   default -> sinks -> stdout not found!
   get rsp msg: Hello, fiber, idx=5
   get rsp msg: Hello, fiber, idx=7
   get rsp msg: Hello, fiber, idx=4
   get rsp msg: Hello, fiber, idx=15
   get rpc error: rejected by server token bucket overload control
   get rsp msg: get rpc error: Hello, fiber, idx=9
   get rsp msg: Hello, fiber, idx=2
   get rpc error: get rsp msg: Hello, fiber, idx=18
   rejected by server token bucket overload control
   get rpc error: get rsp msg: rejected by server token bucket overload controlget rsp msg: 
   Hello, fiber, idx=1
   Hello, fiber, idx=8
   get rpc error: rejected by server token bucket overload control
   get rsp msg: Hello, fiber, idx=11
   rejected by server token bucket overload control
   get rpc error: rejected by server token bucket overload control
   get rpc error: rejected by server token bucket overload control
   get rsp msg: Hello, fiber, idx=12
   get rsp msg: Hello, fiber, idx=14
   get rsp msg: Hello, fiber, idx=13
   ```

5. 等待几秒后在重新发送（此时令牌桶已经有足够的令牌了，全部请求都可以被接受）

   ```
   [root@localhost trpc-cpp]# ./bazel-bin/examples/helloworld/test/fiber_client --client_config=./examples/helloworld/test/conf/trpc_cpp_fiber.yaml
   FLAGS_service_name:trpc.test.helloworld.Greeter
   FLAGS_client_config:./examples/helloworld/test/conf/trpc_cpp_fiber.yaml
   default -> sinks -> stdout not found!
   get rsp msg: Hello, fiber, idx=14
   get rsp msg: Hello, fiber, idx=16
   get rsp msg: Hello, fiber, idx=19
   get rsp msg: get rsp msg: Hello, fiber, idx=10
   get rsp msg: Hello, fiber, idx=8
   get rsp msg: Hello, fiber, idx=15
   get rsp msg: Hello, fiber, idx=6
   Hello, fiber, idx=20
   get rsp msg: Hello, fiber, idx=2
   get rsp msg: Hello, fiber, idx=5
   get rsp msg: Hello, fiber, idx=4
   get rsp msg: Hello, fiber, idx=1
   get rsp msg: Hello, fiber, idx=13
   get rsp msg: Hello, fiber, idx=17
   get rsp msg: Hello, fiber, idx=12
   get rsp msg: Hello, fiber, idx=9
   get rsp msg: Hello, fiber, idx=3
   get rsp msg: Hello, fiber, idx=7
   get rsp msg: Hello, fiber, idx=11
   get rsp msg: Hello, fiber, idx=18
   ```

   
