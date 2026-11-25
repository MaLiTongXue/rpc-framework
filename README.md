# 轻量 RPC 框架（学习实现）

一个**最小可用**的 RPC 框架：消费者只依赖接口，就能像调用本地方法一样调用远程服务。

---

## 📌 出处声明（请先读这一段）

> 本仓库是**跟着鱼皮（程序员鱼皮）的 RPC 框架教程第一章「简易版」实现的**，用于学习 RPC 的核心机制，**不是原创框架**。
>
> - 原作者：程序员鱼皮
> - 原始开源仓库：<https://github.com/liyupi/yu-rpc>
> - 课程：手写 RPC 框架（编程导航）
>
> 代码主体来自课程，作者署名（`@author`）保留在各个源文件里。我在这个基础上做了：
> 精简为只保留简易版、补了聚合 `pom.xml`（一条命令构建）、补充了运行说明和下面这些自己的理解笔记。
>
> 如果你想系统学这套东西，建议直接去看原课程；这个仓库的价值在于「我跑通了、我能讲清楚」。

---

## 这个项目做了什么

**能做的：**

- 消费者定义一个接口，框架自动生成代理对象，调用时把请求发到远端
- 服务端维护「服务名 → 实现类」注册表，收到请求后用反射执行对应方法
- 序列化层抽成接口，默认用 JDK 原生序列化，可替换
- 基于 Vert.x 的 HTTP 服务端，用回调模型处理请求

**不能做的（也是我下一步要补的）：**

- 没有注册中心：地址写死在 `http://localhost:8080`
- 没有负载均衡、重试、容错
- 只有 JDK 序列化，没有 SPI 动态扩展
- 用 HTTP 传，没有自定义二进制协议

---

## 架构与调用链路

```
        消费者                                          提供者
┌──────────────────┐                          ┌──────────────────┐
│  UserService     │  只有接口，没有实现类      │  UserServiceImpl │
│  （接口）         │                          │  （真正的实现）    │
└────────┬─────────┘                          └────────▲─────────┘
         │ ① 调用 userService.getUser(user)            │ ⑤ 反射执行
         ▼                                             │
┌──────────────────┐   ② 打包 RpcRequest    ┌──────────┴──────────┐
│ ServiceProxy     │───────────────────────▶│ HttpServerHandler   │
│ （JDK 动态代理）  │                        │ 反序列化→查表→反射    │
└────────┬─────────┘                        └──────────▲──────────┘
         │ ③ 序列化成 byte[]                            │ ④ HTTP 传输
         ▼                                             │
      ┌───────────────────────────────────────────────┘
      │   Vert.x HTTP Server  ⇄  Hutool HttpRequest
```

一次调用经过 8 步：

1. `ServiceProxyFactory.getProxy(UserService.class)` 拿到代理对象
2. `userService.getUser(user)` 被代理拦截，进入 `ServiceProxy.invoke`
3. 把类名、方法名、参数类型、参数值打包成 `RpcRequest`
4. 序列化成 `byte[]`，用 Hutool 发 HTTP POST
5. 服务端 `HttpServerHandler` 收到，反序列化回 `RpcRequest`
6. 从 `LocalRegistry` 按服务名查到实现类 `UserServiceImpl.class`
7. `Method.invoke` 反射执行，结果包成 `RpcResponse`
8. 序列化返回，消费者拿到 `data`

---

## 模块说明

| 模块 | 作用 |
|---|---|
| `yu-rpc-easy` | 框架本体：代理、注册表、序列化、服务端 |
| `example-common` | 公共依赖：`UserService` 接口与 `User` 模型 |
| `example-consumer` | 服务消费者示例 |
| `example-provider` | 服务提供者示例 |

---

## 快速开始

环境：JDK 17+、Maven 3.6+

```bash
# 1. 在仓库根目录构建全部模块
mvn clean install -DskipTests
```

2. 用 IDEA 打开项目，按顺序运行两个 `main`：

```
example-provider/.../EasyProviderExample.java     ← 先启动，监听 8080，窗口别关
example-consumer/.../EasyConsumerExample.java     ← 再运行
```

3. 控制台打印出 `yupi`，说明整条链路通了。

---

## 核心代码导读

只有 4 个文件需要重点看：

| 文件 | 干什么 | 关键点 |
|---|---|---|
| `proxy/ServiceProxy.java` | 消费端核心 | `implements InvocationHandler`，所有方法调用都落到 `invoke` |
| `proxy/ServiceProxyFactory.java` | 生成代理 | `Proxy.newProxyInstance`，运行期造一个实现接口的类 |
| `server/HttpServerHandler.java` | 服务端核心 | 反序列化 → `LocalRegistry.get` → `Method.invoke` |
| `registry/LocalRegistry.java` | 服务注册表 | `ConcurrentHashMap<服务名, Class>`，服务端多线程安全 |

服务端那三行是整个框架的心脏：

```java
Class<?> implClass = LocalRegistry.get(rpcRequest.getServiceName());
Method method = implClass.getMethod(rpcRequest.getMethodName(), rpcRequest.getParameterTypes());
Object result = method.invoke(implClass.newInstance(), rpcRequest.getArgs());
```

---

## 我的理解笔记

**1. 为什么非要一张 `LocalRegistry`？**

请求传过来的是**字符串**类名，字符串不是类。而 `Class.forName("...UserService")` 拿到的是接口，接口没有方法体，调不了。真正要的是实现类，这个「接口 → 实现类」的映射只有开发者知道，所以要提前 `register` 登记。这一张表的存在，让「新增一个服务」只要一行 register，框架代码不用改。

**2. 动态代理的价值在哪？**

消费者手里只有接口。`Proxy.newProxyInstance` 在运行期生成了一个实现该接口的类，所有方法调用都转到 `invoke(proxy, method, args)`。我在 `invoke` 里能从 `method` 取到类名、方法名、参数类型，从 `args` 取到参数值——四样凑齐了才能发一个请求。

对比静态代理：一个接口就要手写一个代理类（仓库里 `UserServiceProxy` 就是），十个接口十个类；动态代理框架只写一份。

**3. `ConcurrentHashMap` 不是随手选的**

服务端是多线程并发处理请求的，每个请求都要查表。`HashMap` 在并发扩容时会出现结构问题，所以必须用 `ConcurrentHashMap`。

---

## 已知不足与后续计划

- [ ] 注册中心：把写死的地址换成「服务名 → 地址列表」，支持多实例
- [ ] 负载均衡：随机 / 轮询 / 一致性哈希
- [ ] 重试与容错：FailFast / FailOver / FailSafe / FailBack
- [ ] SPI 动态扩展：序列化器、负载均衡器可插拔
- [ ] 自定义二进制协议：替代 HTTP，减少报文头开销
- [ ] 服务端对象复用：现在每个请求都 `newInstance()`，改成单例或交给 Spring 容器

这些正是从「能调通」到「能用」之间缺的部分，也是这个项目目前和 Dubbo 的差距所在。

---

## 环境

- JDK 17+（开发时用 JDK 21/25 编译通过）
- Maven 3.6+
- 依赖：Vert.x 4.5.1、Hutool 5.8.16、Lombok 1.18.30
