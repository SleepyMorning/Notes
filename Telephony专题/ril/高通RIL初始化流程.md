# 高通RIL初始化流程
@(Qcril)[高通][rild]

`Qcril`是高通平台`rild`的平台RIL，是连接Java Telephony Frameworks和Modem的中间传输层。
rild是由C/C++编写的，在使用socket和HIDL等通信方式与其他模块通信。

由于我们主要是介绍Qcril，因此只对rild进行简要描述。

```cpp
//@rild.c

int main(int argc, char **argv) {
    ...
    //打开动态链接库 rilLibPath
    dlHandle = dlopen(rilLibPath, RTLD_NOW);
    ...
    //获取Qcril初始化入口 RIL_Init
    rilInit = (const RIL_RadioFunctions *(*)(const struct RIL_Env *, int, char **))
        dlsym(dlHandle, "RIL_Init");
    ...
    //调用 RIL_Init 完成模块初始化，并返回一个RIL_RadioFunctions结构体
    funcs = rilInit(&s_rilEnv, argc, rilArgv);
    ...
    //将返回的结构体保存在本地
    RIL_register(funcs);
    ...
}
```

我们找到`s_rilEnv`的定义，这个结构体封装了一组方法，它们是用来在消息处理完成后被回调，向Java Telephony Frameworks返回请求结果的重要方法，具体用法放在参考[^s_rilEnv]

```cpp
//@rild.c
static struct RIL_Env s_rilEnv = {
    RIL_onRequestComplete,
    RIL_onUnsolicitedResponse,
    RIL_requestTimedCallback,
    RIL_onRequestAck
};
```

```cpp
//@ril.h
struct RIL_Env {
    void (*OnRequestComplete)(  \
        RIL_Token t,  \
        RIL_Errno e,  \
        void *response,  \
        size_t responselen);

    void (*OnUnsolicitedResponse)(  \
        int unsolResponse,  \
        const void *data,  \
        size_t datalen,  \
        RIL_SOCKET_ID socket_id);


    void (*RequestTimedCallback) (  \
        RIL_TimedCallback callback,  \
        void *param,  \
        const struct timeval *relativeTime);

    void (*OnRequestAck) (RIL_Token t);
};
```

## RIL_Init - 初始化

我们找到对应的`RIL_Init`函数，

```cpp
//@ril_api.cpp

extern "C"const RIL_RadioFunctions * RIL_Init(const struct RIL_Env * env, int argc, char * *argv) {
    ...
    //QCRIL_DEFAULT_INSTANCE_ID值为0，QCRIL_SECOND_INSTANCE_ID值为1
    qcril_response_api[QCRIL_DEFAULT_INSTANCE_ID] = (struct RIL_Env * ) env;
    qcril_response_api[QCRIL_SECOND_INSTANCE_ID] = (struct RIL_Env * ) env;
    ...

    //执行android_ril_module的ril_init
    get_android_ril_module().ril_init(instance_id, env, argc, c_argv);

    //返回qcril_request_api
    return & qcril_request_api[QCRIL_DEFAULT_INSTANCE_ID];
} /* RIL_Init() */
```

我们先看下`qcril_response_api`和`qcril_request_api`的定义。原来两者都是定义为数组，区别是前者需要rild传入参数进行赋值（就是文章开头说的那个`s_rilEnv`），后者则是内置的。这样设计是比较容易理解的，RilRequst的处理是在内部，而如何将处理结果传送回Java层由外部方法来处理。

```cpp
//@ril_api.cpp

static RIL_Env *qcril_response_api[QCRIL_MAX_INSTANCE_ID];

static const RIL_RadioFunctions qcril_request_api[] = {
    {
        QCRIL_RIL_VERSION,
        onRequest,
        currentState_rid,
        onSupports_rid,
        onCancel_rid,
        getVersion_rid
    }
};
```

接下来是`ril_init`，首先需要弄清楚到底是谁在调用这个函数。我们找到`get_android_ril_module`定义的地方，

```cpp
//@Android_ril_module.cpp

//定义局部变量 the_module
static load_module<android_ril_module> the_module;

android_ril_module &get_android_ril_module() {
    return the_module.get_module();
}
```

 `load_module`看上去是一个模板，

```cpp
//@Module.h

template <typename M>
class load_module
{
    public:
        load_module() {
            get_module().init();
        }
        M &get_module() {
            static M module{};
            return module;
        }
};
```

由此可得出`the_module`的类型是`load_module`，其定义形式应该是下面这样：

```cpp
class load_module
{
    public:
        load_module() {
            get_module().init();
        }
        android_ril_module &get_module() {
            static android_ril_module module{};
            return module;
        }
};
```

所以`get_android_ril_module()`返回值其实是定义在局部变量`the_module`的`get_module`函数中全局变量`module`，其类型为`android_ril_module`。

因此`get_android_ril_module().ril_init(instance_id, env, argc, c_argv);`的真实形式是`android_ril_module`中定义的`ril_init`。

```cpp
//@Android_ril_module.cpp

void android_ril_module::ril_init
(
    qcril_instance_id_e_type instance_id,
    const struct RIL_Env *env,
    int argc,
    char **argv
)
{
    //将env保存到本地，并替换掉OnRequestComplete和OnRequestAck
    local_env = *env;
    local_env.OnRequestComplete = ::local_on_request_complete;
    local_env.OnRequestAck = ::local_on_request_ack;

    //legacy_RIL_Init才是qcril的实际入口，返回的结构体被
    //赋值给qcril_legacy_functions而没有直接返回给rild，这是
    //高通做的适配层，这样做好处多多，自行体会吧。
    qcril_legacy_functions[instance_id] = legacy_RIL_Init(&local_env, instance_id, argc, argv);

    std::shared_ptr<QcrilInitMessage> qcril_init_msg = std::make_shared<QcrilInitMessage>(instance_id);
    if(qcril_init_msg) {
      qcril_init_msg->broadcast();
    }
}
```

我们继续看下`QcrilInitMessage`的定义，

```cpp
class QcrilInitMessage: public UnSolicitedMessage,
        public add_message_id<QcrilInitMessage>
```

又是使用了模板，这次`add_message_id<QcrilInitMessage>`直接展开，

```cpp
class add_message_id
{
    public:
        static inline message_id_t get_class_message_id() {
            //Dispatcher是单例模式编写
            static message_id_t id = Dispatcher::getInstance()
                .registerMessage("com.qualcomm.qti.qcril.core.qcril_init")
            return id;
        }
};
```

```cpp
//@Dispatcher.cpp

message_id_t Dispatcher::registerMessage(const string &msg_name)
{
    lock_guard<recursive_mutex> lock(mMutex);
    message_id_info *h = nullptr;
    for (auto &mh : mMessageHandlers ) {
        const std::string &name = mh.get_name();
        if (name == msg_name) {
            h = &mh;
            break;
        }
    }
    if (h == nullptr) {
        auto it = mMessageHandlers.insert(mMessageHandlers.end(), message_id_info{msg_name});
        h = &*it;
        h->idx = mMessageHandlers.size() - 1;
    }
    return new message_id_info(*h);
}
```

```cpp
//@Mesage_id.h

typedef struct message_id_info *message_id_t;

struct message_id_info
{
    friend class Dispatcher;
    private:
        std::shared_ptr<std::string> m_name;
        std::vector<Module*> module_list;

    public:
        size_t idx = 0;
}
```


## RIL_register - 注册
看下Qcril初始化完成返回的结构体是什么和这个结构体被用来做什么。

我们打开`RIL_RadioFunctions`的定义，就知道里面封装了4个函数和一个版本信息。

```cpp
//@ril.cpp

typedef struct {
    int version;        /* 当前ril的版本号 */
    RIL_RequestFunc onRequest;  /*函数指针，用于处理RilRequest*/
    RIL_RadioStateRequest onStateRequest;  /*用于查询Radio状态*/
    RIL_Supports supports;  /*是否支持某个RilRequest，1为支持，0为不支持*/
    RIL_Cancel onCancel;  /*取消一个未处理的RilRequest*/
    RIL_GetVersion getVersion;  /*获取当前ril实现的版本号*/
} RIL_RadioFunctions;
```

```cpp
//@ril.h

typedef void * RIL_Token;

typedef void (*RIL_RequestFunc) ( \
    int request, \     /*命名为RIL_REQUEST_*的请求ID*/
    void *data, \      /*请求中携带的数据*/
    size_t datalen, \  /*请求中携带数据的长度*/
    RIL_Token t, \
    RIL_SOCKET_ID socket_id);

typedef RIL_RadioState (*RIL_RadioStateRequest)( \
    RIL_SOCKET_ID socket_id);

typedef int (*RIL_Supports)(int requestCode);

typedef const char * (*RIL_GetVersion) (void);
```

我们再看下RIL_RadioFunctions是怎么被处理的，找到`RIL_register`的函数，
```cpp
//@ril.cpp

extern "C" void RIL_register (const RIL_RadioFunctions *callbacks) {
    ...
    //保存到全局变量s_callbacks，这样其他地方也可以使用到
    memcpy(&s_callbacks, callbacks, sizeof (RIL_RadioFunctions));
    //标识s_callbacks已经被赋值
    s_registerCalled = 1;
    ...
    //使用radio命名空间下的registerService进行注册
    radio::registerService(&s_callbacks, s_commands);
}
```

```cpp
@ril_service.cpp

void radio::registerService(RIL_RadioFunctions *callbacks, CommandInfo *commands) {
    using namespace android::hardware;
    int simCount = 1;
    const char *serviceNames[] = {
            android::RIL_getServiceName(), RIL2_SERVICE_NAME};

    simCount = SIM_COUNT;

    configureRpcThreadpool(1, true /* callerWillJoin */);
    for (int i = 0; i < simCount; i++) {
        pthread_rwlock_t *radioServiceRwlockPtr = getRadioServiceRwlock(i);
        int ret = pthread_rwlock_wrlock(radioServiceRwlockPtr);
        assert(ret == 0);

        radioService[i] = new RadioImpl;
        radioService[i]->mSlotId = i;
        oemHookService[i] = new OemHookImpl;
        oemHookService[i]->mSlotId = i;
        RLOGD("registerService: starting android::hardware::radio::V1_1::IRadio %s",
                serviceNames[i]);
        android::status_t status = radioService[i]->registerAsService(serviceNames[i]);
        status = oemHookService[i]->registerAsService(serviceNames[i]);

        ret = pthread_rwlock_unlock(radioServiceRwlockPtr);
        assert(ret == 0);
    }

    s_vendorFunctions = callbacks;
    s_commands = commands;
}
```

---

[^s_rilEnv]: s_rilEnv。