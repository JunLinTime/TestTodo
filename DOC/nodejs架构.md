# 架构

## 体系架构
Node.js 主要分为四大部分
- Node Standard Library: 标准库，http, Buffer 模块，由Javascript编写的
- Node Bingdings： JS 和 C++的桥梁，封装V8 和 Libuv的细节，向上层提供基础API服务
- V8： google开发的JavaScript引擎，提供JavaScript运行环境，是Node.js的发动机
- Libuv：提供跨平台的异步 I/O 能力
- C-ares: 提供了异步处理 DNS 相关的能力
- http_parser、OpenSSL、zlib等：提供包括http解析，SSL，数据压缩等其他的功能


## 为啥是libuv

| 分类 | 操作 | 时间成本 |
| --- | --- | --- |
| 缓存 | L1 缓存 | 1ns |
| | L2 缓存 | 4ns |
| | 主存储器 | 100ns |
| | SSD 随机读取 | 16000ns |
| IO | 往返在同一数据中心 | 500000ns |
| | 物理磁盘寻道 | 4000000 ns |

即便是SSD的访问相对于高速的CPU，仍然是慢速设备。基于

## V8

JS 引擎的执行过程大致是 源代码 -> 抽象语法树 -> 字节码 -> JIT -> 本地代码

**Isolate**
一个Isolate是一个独立的虚拟机。对应一个或多个线程，但同一时刻只能被一个线程进入。所有的Isolate是完全独立的，它们不能够有任何共享的资源。

Context: 上下文环境
Scope: 作用域
handle: 句柄

**Handle概念**

V8 内存分配在Heap中分配的

GC 需要对 V8 中的所有对象进行跟踪，而对象都是用 Handle 方式引用的，GC需要对 Handle 进行管理，
这样GC就能知道 Heap中一个对象的引用情况，当一个对象的Handle引用发生改变的时候，GC 即可对该对象进行回收或者移动。

因此，V8编程必须使用Handle去引用一个对象，而不是直接通过C++的方式获取对象的引用。

Handle 分为 Local 和 Persistent 两种

Local 局部的，同时被 HandleScope 进行管理。
Persistent 全局的，不受 HandleScope的管理，其作用域可以延伸到不同的函数。

Persistent Handle对象需要 Persistent::New, Persistent::Dispose 配对使用，类似于 C++ 的 new 和 delete

**Scope**

从概念上讲，这个上下文环境也可以理解为运行环境，在执行 javascript 脚本的时候，总要有一些环境变量或者全局函数。我们如果要在自己的 C++ 代码中嵌入 v8 引擎，
让其他用户从脚本中直接调用，这样才会体现出 javascript 的强大，我们可以用 C++ 编写全局函数或者类，让其他人通过 javascript 进行调用，这样就无形中扩展了 javascript 的功能。

Context 嵌套。以最近的为准。

context 内可以建立 scope 用来容纳其他 context

**关系**

1. Handle<Context> context = Context::New();   Persistent handle
2. Context::Scope context_scope(context);
3. HandleScope handle_scope;
4. Handle<String> source_obj = String::New(argv[[]1]);
5. Handle<Script> script_obj = Script::Compile(source_obj);
6. Handle<Value> local_result = script_obj->Run();
7. context.Dispose();

## C++ 和 JS 交互

数据及模板：

由于C++原生数据类型与JavaScript中数据类型有很大差异，因此V8提供了 Value 类，从 JavaScript 到 C++, 从 C++ 到 JavaScript 都会用到这个类及其子类

```js
Handle<Value> Add(const Arguments& args) {
    int a = args[0]->Uint32Value();
    int b = args[1]->Uint32Value();

    return Integer::New(a+b)
}
```

Integer 即为 Value的一个子类。
V8中，有两个模板（Template）类（并非 C++ 中的模板类）
对象模板（ObjectTemplate）
函数模板（FunctionTemplate）

**JS 使用 C++ 变量**
在 JavaScript 与 V8 间共享变量事实上是非常容易的吗，基本模板

```c++
static char sname[512] = {0};

static Handle<Value> NameGetter(Local<String> name, const AccessorInfo& info) {
    return String::New((char*)&name, strlen((char*)&name));
}

static void NameSetter(Local<String> name, Local<Value> value, const AccessorInfo& info) {
    Local<String> str = value->ToString();
    str->WriteAscii((char*)&sname);
}
```

定义了 NameGetter, NameSetter 之后，在main函数中，将其注册在 global 上：

```c++
// Create a template for the global object
Handle<ObjectTemplate> global = ObjectTemplate::New();
// public the name variable to script
global->SetAccessor(string::New("name"), NameGetter, NameSetter);
```

**JS 使用 C++ 函数**

极大程度的增强JavaScript脚本读写的能力，如文件读写，网络/数据库访问，图形/图像处理，类似于 JAVA 的 jni 技术。

在 C++ 代码中，定义以下原型的函数：

```c++
Handle<Value> func(const Arguments& args) {}// return something
```

然后将其公开给脚本：
```
global->Set(String::New("func"), FunctionTemplate::New(func));
```

**JS 使用C++类**

如果从面向对象的视角来分析，最合理的方式是将 C++ 类公开给 JavaScript，这样可以将 Javascript 内置的对象数量大大增加。

最好的场景：既有脚本语言的灵活性，又有 C/C++ 等系统语言的效率。使用 V8 引擎，可以很方便的将 C++ 类包装成 JS 使用的资源。

举例：

```c++
class Person {
    private:
        unsigned int age;
        char name[512];

    public:
        Person(unsigned int age, char *name) {
            this->age = age;
            strncpy(this->name, name, sizeof(this->name));
        }

        unsigned int getAge() {
            return this->age;
        }

        void setAge(unsigned int nage) {
            this->age = nage
        }

        char *getName() {
            return this->name;
        }

        void setName(char *nname) {
            strncpy(this->name, nname, sizeof(this->name));
        }
}

```

定义构造器的包装

```c++
Handle<Value> PersonConstructor(const Arguments& args){
    Handle<Object> object = args.This();
    HandleScope handle_scope;
    int age = args[0]->Uint32Value();

    String::Utf8Value str(args[1]);
    char* name = ToCString(str);

    Person *person = new Person(age, name);
    object->SetInternalField(0, External::New(person));
    return object;
}
```
从args中获取参数并转换为合适的类型之后，我们根据此参数来调用Person类实际的构造函数，并将其设置在 object 的内部字段中，紧接着需要包装 Person 类的 getter/setter:

```c++
Handle<Value> PersonGetAge(const Arguments& args){
    Local<Object> self = args.Holder();
    Local<External> wrap = Local<External>::Cast(self->GetInternalField(0));

    void *ptr = wrap->Value();
    
    return Integer::New(static_cast<Person*>(ptr)->getAge());
}

Handle<Value> PersonSetAge(const Arguments& args){
    Local<Object> self = args.Holder();
    Local<External> wrap = Local<External>::Cast(self->GetInternalField(0));

    void* ptr = wrap->Value();
    
    static_cast<Person*>(ptr)->setAge(args[0]->Uint32Value());
    return Undefined();
}
```

上面相对于对 getAge setAge 的函数包装，完成后需要将 Person 类暴露给脚本环境：

首先，创建一个新的函数模板，将其与字符串 "Person" 绑定，并放入 global:

```c++
Handle<FunctionTemplate> person_template = FunctionTemplate::New(PersonConstructor);
person_template->SetClassName(String::New("Person"));
global->Set(String::New("Person"), person_template);
```
定义原型模板
```c++
Handle<ObjectTemplate> person_proto = person_template->PrototypeTemplate();

person_proto->Set("getAge", FunctionTemplate::New(PersonGetAge))
person_proto->Set("setAge", FunctionTemplate::New(PersonSetAge)); 

person_proto->Set("getName", FunctionTemplate::New(PersonGetName)); 
person_proto->Set("setName", FunctionTemplate::New(PersonSetName));
```

最后设置实例模板：
```c++
Handle<ObjectTemplate> person_inst = person_template->InstanceTemplate(); 
person_inst->SetInternalFieldCount(1);
```

**C++调用JS函数**

我们直接看下 src/timer_wrap.cc 的例子，V8 编译执行了 timer.js 构造了 Timer 对象

```c++
static void OnTimeout(uv_timer_t* handle) {
    TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
    Environment* env = wrap->env();
    HandleScope handle_scope(env->isolate());
    Context::Scope context_scope(env->context())
    wrap->MakeCallback(kOnTimeout, 0, nullptr) 
}

inline v8::Local<v8::Value> AsyncWrap::MakeCallback(uint32_t index, int argc, v8::Local<v8::Value>* argv) {
    v8::Local<v8::Value> cb_v = object()->Get(index);
    CHECK(cb_v->IsFunction());
    return MakeCallback(cb_v.As<v8::Function>(), argc, argv);
}
```
TimerWrap 对象通过数组的索引寻址，找到 Timer 对象索引 0 的对象，而对其赋值的是在 lib/timer.js 里面的 list._timer[kOnTimeout] = listOnTimeout;
这边找到的对象是个 Function, 后面忽略 domains 异常处理等，就是简单的调用 Function 对象的 Call 方法, 并且传入上文提到的 Context 和参数。

```c++
Local<Value> ret = callback0->Call(recv, argc, argv)
```
这就实现了 C++ 对 JS函数的调用

