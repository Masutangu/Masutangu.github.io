---
layout: post
date: 2016-08-30T08:14:24+08:00
title: 简单异步应用框架的实现
tags: 个人项目
---

两年前刚进公司的时候，第一次接触了异步框架，那时还处于懵懵懂懂的状态。最近换了组，接触到另外一种实现的异步框架，这次有了一定的积累后，对异步框架的设计也有了更多的理解。刚好最近自己基于 libuv 造了个简单的轮子 [saf (Simple Async Framework)](https://github.com/Masutangu/SAF)，趁此机会和大家聊聊异步框架的设计思想和实现。

# 异步框架设计思想
## 服务器模型
先来看看传统的服务器模型，如下图：

<img src="/assets/images/simple-async-framework/illustration-1.png" width="800" />

一般来说，服务器端可以分为三层：**接入层**，**逻辑层**，**数据层**。接入层负责客户端的接入，逻辑层则实现业务逻辑，数据层就是数据的存储。

简单来说，逻辑层做的事情无非就是解析客户端的请求包，写入数据到数据层或从数据层读取数据，再组装回包发送给客户端。

我们拿微博做例子，用户登录微博，客户端发起拉取首页的请求，server 首先解析客户端请求，拿到用户 id，再根据用户 id 到数据层查询以下数据并拼装回包发回给客户端：

* 关注数
* 粉丝数
* 微博数
* 个人简介，包括头像
* 微博时间轴，即关注的人最近发的微博

## 同步 vs 异步
继续上面微博的例子，我们假设微博时间轴采用拉的方式去获取。
同步的 server，实现的逻辑如下图：

<img src="/assets/images/simple-async-framework/illustration-2.png" width="800" />

如果同步的 server 是单线程，那每次发送请求到数据层查询数据时都会阻塞，在收到数据层的回包前 server 做不了其他事情，CPU 在等待期间空转，非常浪费资源。

异步 server 则不会有这个烦恼，当 server 向数据层发送请求时会立即返回，此时 server 可以处理其它客户端请求，直到数据层返回所请求的数据，通知到 server，server 再继续之前的业务逻辑。流程图大致如下：

<img src="/assets/images/simple-async-framework/illustration-3.png" width="800" />

我们再仔细看上面的流程图，可以发现除了拉取微博时间轴需要依赖关注人列表之外，其它数据查询都互不依赖。因此可以把流程优化下：

<img src="/assets/images/simple-async-framework/illustration-4.png" width="800" />

通过这样的优化，耗时从 

**关注数请求耗时 + 粉丝数请求耗时 + 微博数请求耗时 + 个人简介耗时 + 时间轴耗时**

缩减到 

**MAX(关注数请求耗时，粉丝数请求耗时，微博数请求耗时，个人简介耗时) + 时间轴耗时**。

## 模型抽象化
通过上面的例子，来讲讲如何将上述异步处理逻辑抽象化。
我们可以把业务逻辑以**状态（state）**为单位来划分，如下图。**状态与状态之间是串行的**，即你必须执行完一个状态，才会跳转到下一个状态。比如我们必须先拉取关注列表，才能根据关注列表去拉取时间轴。

<img src="/assets/images/simple-async-framework/illustration-5.png" width="800" />

而一个状态内可以有很多**动作（action）**，**一个状态内的动作是互相不依赖的，即可以并行执行**，如下图。如我们可以同时发请求拉去关注数，粉丝数，微博数，因为他们之间是互相独立没有依赖的。

<img src="/assets/images/simple-async-framework/illustration-6.png" width="800" />


# 异步框架的实现
讲完了概念，开始来实践。linux 下有 epoll 模型，另外还有大名鼎鼎的 libuv 提供了跨平台的异步 IO。那接下来结合我自己造的轮子，谈谈如何基于 epoll 或 libuv 来实现一个异步框架。

## 状态保存
无论是函数调用，或者线程切换，都会保存上下文，等到函数调用返回或线程切回来时，才能继续处理之前未完成的逻辑。而我们的异步模型（其实就是状态机），也是类似的道理，我们需要在请求发送时保存好上下文，才能在收到回包时继续之前的逻辑往下走。
saf 是基于 libuv 的，因此我使用 libuv 的 handle 结构体的 data 字段来保存上下文。如果是直接使用 epoll 来实现异步server，则可以用 fd 来绑定上下文（全局的 map，key 为 fd，value 为上下文信息）。

## 消息透传
既然各个状态是有依赖关系的，那就得有一个消息（message）实体贯穿整个处理流程。通过这个消息实体来传递各个状态所需要的信息。这也是为什么 saf 中 action 和 state 的接口都有一个 msg 参数的原因（见下节**接口设计**）。

## 接口设计
封装一个异步框架，意味着对于框架使用者来说其无需关心网络收发包的细节，只需关心自身业务逻辑的实现。那我们在设计接口上就需要屏蔽这些细节。

既然要对使用者屏蔽收发包细节，表明收包和发包的回调都由框架来控制。因此我们只需要暴露打包请求包和解包回包的接口给使用者去实现。框架调用使用者实现的打包接口后，将打好的 buffer 发送出去，在收到回包之后，再调用使用者实现的解包接口来处理回包。

在 saf 的接口设计中，我尽量保持接口命名的统一，```prepareProcess``` 表示在执行前的预处理工作，```afterProcess``` 表示执行完后的后续处理工作。下面可以看到在不同的模块中，```prepareProcess``` 和 ```afterProcess``` 的功能略有不同。

### 消息类
```c++
//msg.h

class Msg {
public:
    virtual ~Msg() {}
};
```
如上所述，消息用于状态之间传递依赖的信息，由业务自行继承添加所需成员。

### Handler 类
```c++
    //handler.h

    /*
     * 解析客户端请求包
     * 返回 > 0 表示收包不完整
     * 返回 0 表示解析成功
     * 返回 < 0 表示解包失败, server将会杀掉客户端连接
     */
    virtual int prepareProcess(char* buf, unsigned int len, Msg* msg) = 0;

    /*
     * 打包客户端回包到输入 buf 中,len 为输入 buf 长度
     * 返回 > 0 表示 buf 不够, len 为实际需要的 buf 长度
     * 返回 0 表示打包成功, len 为 buf 的实际长度
     * 返回 < 0 表示打包失败, server将会杀掉客户端连接
     */
    virtual int afterProcess(char* buf, unsigned int& len, Msg* msg) = 0;

    /*
     * 创建该 handler 的 msg
     */
    virtual Msg* createMsg() = 0;
```
Handler 类对应客户端请求的处理流程。业务继承 Handler 基类，实现请求包和回包的打解包接口以及创建业务消息的接口。

在 Handler 的构造函数添加该 Handler 包含的 State。在收到客户端请求后，框架调用相应的 Handler 的 ```prepareProcess``` 接口对客户端请求进行解包。然后依次执行各个 State，全部 State 执行完成后，框架调用该 Handler 的 ```afterProcess``` 将回包打包到传入的 buffer 参数，再由框架将该 buffer 发送回客户端。

### State 类

```c++
    // state.h

    /*
     * 执行 state 包含的 action 前, 框架会调用该函数, 可以做预处理工作
     * 返回 0 表示成功
     * 返回 != 0 表示失败
     */
    virtual int prepareProcess(Msg* msg) { return 0; };
    /*
     * state 包含的 action 都执行完时, 框架会调用该函数,可以做一些后续处理工作
     * 返回 0 表示成功
     * 返回 != 0 表示失败
     */
    virtual int afterProcess(Msg* msg) { return 0; };
```
State 类对应上面**模型抽象化**小节的**状态**。

在 State 的构造函数添加该 State 包含的 Action。State 执行前，框架调用该 State 的 ```prepareProcess``` 接口，使用者可以在该接口做些预处理工作。当 State 执行完成后，框架调用该 State 的 ```afterProcess``` 接口。


### Action 类

```c++
    //action.h 
    
    /*
     * 设置 action 的目的 ip，端口和通信协议（目前只支持tcp） 
     */
    void setActionInfo(const std::string& ip, int port, int protocol);
    
    /*
     * 设置 action 的超时时间，单位为毫秒。 <=0 为永不超时
     */
    void setTimeout(unsigned int timeout) { m_timeout = timeout; }  

    /*
     * 打包 Action 请求包到输入 buf 中, len 为输入 buf 的长度
     * 返回 0 表示打包成功, len 为实际需要的 buf 长度
     * 返回 > 表示 buf 不够, len 为实际需要的 buf 长度
     * 返回 < 0 表示失败
     */
    virtual int prepareProcess(char* buf, unsigned int& len, Msg* msg) = 0;

    /*
     * 解析 Action 回包
     * 返回 0 表示解析回包成功
     * 返回 < 0 表示出错
     * 返回 > 0 表示收包未完整
     */
    virtual int afterProcess(char* buf, unsigned int len, Msg* msg) = 0;
```
Action 类对应上面**模型抽象化**小节的**动作**。

Action 执行前，框架调用该 Action 的 ```prepareProcess``` 接口，将 Action 的请求包打包到传入的 buffer 参数，当收到 Action 的回包后，框架会调用 Action 的 ```afterProcess``` 接口，将回包解包。

### REGISTER_HEADER_PARSER
```REGISTER_HEADER_PARSER``` 宏用于注册解析请求包头函数。

### REGISTER_HANDLER
```REGISTER_HANDLER``` 宏用于注册请求对应的 handler 类

## 状态机逻辑
接下来看看 saf 如何将 handler／state／action 串联起来（代码有所简化）

```c++
    //handler.cpp
    /* 
     * 主逻辑，ClientContext 保存了客户端会话的上下文
     * 其 m_state_idx 成员表示当前所属的状态 id
     * m_action_idx 成员表示处于当前所属状态的动作id
     * m_msg 即业务定义的消息类，被透传给 state 和 action 中
     */  
    void Handler::process(ClientContext* c_ctx) {
        if (c_ctx->m_state_idx < m_state_list.size()) {
            State* state = m_state_list[c_ctx->m_state_idx++];
            // 执行 state 前, 将 action_idx 置 0
            c_ctx->m_action_idx = 0;
            // 调用 state 的 prepareProcess 接口
            state->prepareProcess(c_ctx->m_msg);
            // 开始执行该state
            state->process(c_ctx);   
        } else {
            static char buf[DEFAULT_BUF_SIZE];
            char* actual_buf = buf;
            unsigned int actual_len = DEFAULT_BUF_SIZE;
            // 调用 handler 的 afterProcess 接口，打包回包到 actual_buf 中
            afterProcess(actual_buf, actual_len, c_ctx->m_msg);
            // 发送回包给客户端
            c_ctx->sendResponse(actual_buf, actual_len);
        }
    }
```

```c++
    //state.cpp

    /* 
     * state 处理逻辑
     */
    void State::process(ClientContext* c_ctx) {
        // 如果没有action,直接finish
        if (m_action_list.size() == 0) {
            finish(c_ctx);
            return;
        }

        static char buf[DEFAULT_BUF_SIZE];
        char* actual_buf = NULL;
        unsigned int actual_len = 0; // buf 的实际长度
        int ret = 0;

        // 执行该 state 下所有 action 
        for(unsigned int i = 0; i < m_action_list.size(); i++) {
            Action* action = m_action_list[i];
            actual_len = DEFAULT_BUF_SIZE;
            actual_buf = buf;
            // 调用 action 的 prepareProcess 接口，打包 action 的请求包到 actual_buf 中
            action->prepareProcess(actual_buf, actual_len, c_ctx->m_msg);
            // 执行该 action
            c_ctx->processAction(action, actual_buf, actual_len);
        }
    }

    // action 完成后回调该接口，如果所有action都完成，则调用下面的 finish 接口
    void State::finishAction(ClientContext* c_ctx) {
        printf("finishAction\n");
        c_ctx->m_action_idx++;
        if (c_ctx->m_action_idx >= m_action_list.size()) {
            finish(c_ctx);
        }
    }

    // 调用 state 所属的 handler 的 process 函数
    void State::finish(ClientContext* c_ctx) {
        afterProcess(c_ctx->m_msg);
        m_handler->process(c_ctx);
    }
```

```c++
    //ClientContext.cpp

    /*
     * action 收到回包后的回调
     */
    static void recvActionRsp(uv_stream_t *server, ssize_t nread, const uv_buf_t *buf) {
        // data 字段保存了 action 的上下文
        ActionContext* a_ctx = (ActionContext*) server->data;
        a_ctx->recv_buf.append(buf->base, nread);
        // action 的上下文中保存了客户端请求的上下文
        ClientContext* c_ctx = a_ctx->c_ctx;
        // 调用 action 的 afterProcess 接口
        int ret = a_ctx->action->afterProcess(a_ctx->recv_buf.data(), a_ctx->recv_buf.len(), c_ctx->m_msg);
        // 通知 action 所属的 state 该 action 完成了
        a_ctx->action->m_state->finish(a_ctx->c_ctx);
    }
```

## 样例
以下是 saf 的一个简单的 [demo](https://github.com/Masutangu/SAF/blob/master/sample.cpp)。代码仅说明用，所以比较简单粗暴。该 server 的所有请求都由 myHandler 来处理，myHandler 包含一个状态 myState1。myState1 包含一个 Action, 该 Action 将客户端请求包拷贝并通过 tcp 发送给 127.0.0.1:7000 的服务，接收到回包后再把回包原样发回给客户端。  

```c++
//
// Created by Masutangu on 16/8/9.
//

#include "saf/header.h"

#include <cstring>

using namespace saf;

const int BUF_SIZE = 1024;

class myMsg: public Msg {
public:
    char readbuf[BUF_SIZE];
    char writebuf[BUF_SIZE];
};

class myAction: public Action {
public:
    int prepareProcess(char* buf, unsigned int& len, Msg* msg);
    int afterProcess(char* buf, unsigned int len, Msg* msg);
};

int myAction::prepareProcess(char* buf, unsigned int& len, Msg* msg) {
    myMsg* my_msg = static_cast<myMsg*> (msg);
    printf("myAction prepareProcess, data: %s\n", my_msg->readbuf);
    if (len >= BUF_SIZE) {
        memcpy(buf, my_msg->readbuf, BUF_SIZE);
        return 0;
    } else {
        len = BUF_SIZE;
        return BUF_SIZE;
    }

}

int myAction::afterProcess(char* buf, unsigned int len, Msg* msg) {
    printf("myAction afterProcess: %s\n", buf);
    myMsg* my_msg = static_cast<myMsg*> (msg);
    memcpy(my_msg->writebuf, buf, len < BUF_SIZE ? len:BUF_SIZE);
    return 0;
}

class myState1: public State {
public:
    myState1();
};

myState1::myState1() {
    myAction* action = new myAction;
    action->setActionInfo("127.0.0.1", 7000, 0); //设置action的ip和端口
    addAction(action);
}

class myHandler: public Handler {
public:
    myHandler();
    Msg* createMsg();
    int prepareProcess(char* buf, unsigned int len, Msg* msg);
    int afterProcess(char* buf, unsigned int& len, Msg* msg);

};

Msg* myHandler::createMsg() {
    return new myMsg();
}

myHandler::myHandler() {
    myState1* state1 = new myState1();
    addState(state1);

}

int myHandler::prepareProcess(char* buf, unsigned int len, Msg* msg) {
    printf("handler: prepareProcess len=%d\n", len);
    myMsg* my_msg = static_cast<myMsg*> (msg);
    memcpy(my_msg->readbuf, buf, len);
    return 0;
}

int myHandler::afterProcess(char* buf, unsigned int& len, Msg* msg) {
    myMsg* my_msg = static_cast<myMsg*> (msg);
    if (len >= 1024) {
        memcpy(buf, my_msg->writebuf, 1024);
        return 0;
    } else {
        len = 1024;
        return 1;
    }
}

int parseReq(char* buf, unsigned int len) {
    return 1; // 该请求的类型为 1，由 myHandler 处理
}

int main() {
    REGISTER_HANDLER(1, myHandler);  // 请求类型为 1 的由 myHandler 类处理
    REGISTER_HEADER_PARSER(parseReq); // 请求包头由 parseReq 函数解析

    AsyncServer server = AsyncServer();
    server.setBindAddress("0.0.0.0", 8000); // 监听 8000 端口
    server.run();
}
```

# 总结
由于时间和能力有限，saf 目前来说非常简陋，也没有经过严格的测试。对于一个框架来说，要做的事情还有很多，比如日志模块的完善、性能分析和优化。不过，**done is better than perfect**. 最后，如有问题或意见，欢迎留言或者 email 我，也欢迎转载分享～


   



