# oneAPI Level Zero 项目深度调研报告

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 仓库目录结构](#2-仓库目录结构)
- [3. 整体软件架构](#3-整体软件架构)
- [4. 核心组件详解](#4-核心组件详解)
  - [4.1 API 头文件层 (include/)](#41-api-头文件层-include)
  - [4.2 库层 (source/lib/)](#42-库层-sourcelib)
  - [4.3 加载器层 (source/loader/)](#43-加载器层-sourceloader)
  - [4.4 层系统 (source/layers/)](#44-层系统-sourcelayers)
  - [4.5 驱动层 (source/drivers/)](#45-驱动层-sourcedrivers)
  - [4.6 工具层 (source/utils/)](#46-工具层-sourceutils)
- [5. 四大 API 子系统](#5-四大-api-子系统)
- [6. DDI (设备驱动接口) 机制](#6-ddi-设备驱动接口-机制)
- [7. 完整调用流程分析](#7-完整调用流程分析)
  - [7.1 初始化流程 (zeInit)](#71-初始化流程-zeinit)
  - [7.2 驱动发现与排序](#72-驱动发现与排序)
  - [7.3 API 调用分发流程](#73-api-调用分发流程)
  - [7.4 层拦截链](#74-层拦截链)
- [8. 关键设计模式](#8-关键设计模式)
- [9. 构建系统](#9-构建系统)
- [10. 测试架构](#10-测试架构)
- [11. 线程安全与错误处理](#11-线程安全与错误处理)
- [12. 环境变量配置](#12-环境变量配置)
- [13. 示例程序分析](#13-示例程序分析)
- [14. 总结](#14-总结)

---

## 1. 项目概述

oneAPI Level Zero 是 Intel oneAPI 生态系统中的**底层硬件抽象层**，提供对 GPU、NPU 等异构计算设备的直接、低开销访问。该仓库（版本 v1.28.2）包含以下核心组件：

| 组件 | 说明 |
|------|------|
| **API 头文件** | Level Zero 规范的 C/C++ 头文件定义 |
| **Loader（加载器）** | 运行时驱动发现、加载和分发 |
| **Validation Layer（验证层）** | 参数校验、句柄生命周期追踪 |
| **Tracing Layer（追踪层）** | API 调用追踪与性能分析 |
| **Null Driver（空驱动）** | 用于测试的模拟驱动 |

**技术定位**：Level Zero 位于应用程序和硬件驱动之间，类似于 Vulkan 之于图形渲染，Level Zero 为通用计算提供了一个薄而高效的抽象层。

---

## 2. 仓库目录结构

```
level-zero/
├── include/                 # 公共 API 头文件（C/C++ + Python 绑定）
│   ├── ze_api.h             #   核心计算 API（18,715 行）
│   ├── ze_ddi.h             #   核心 DDI 函数指针表
│   ├── ze_ddi_common.h      #   通用 DDI 定义
│   ├── zes_api.h            #   系统管理 API（8,809 行）
│   ├── zes_ddi.h            #   系统管理 DDI
│   ├── zet_api.h            #   工具/调试 API（3,972 行）
│   ├── zet_ddi.h            #   工具 DDI
│   ├── zer_api.h            #   运行时辅助 API（162 行）
│   ├── zer_ddi.h            #   运行时 DDI
│   ├── layers/              #   层 API（追踪回调注册）
│   └── loader/              #   加载器公共 API
│       └── ze_loader.h      #     加载器接口
│
├── source/                  # 核心实现代码
│   ├── lib/                 #   库层（用户链接的 libze_loader）
│   │   ├── ze_lib.cpp/h     #     核心库上下文和初始化
│   │   ├── ze_libapi.cpp    #     ZE API 分发入口（16,087 行）
│   │   ├── ze_libddi.cpp    #     DDI 表填充
│   │   ├── zes_libapi.cpp   #     ZES API 分发
│   │   ├── zet_libapi.cpp   #     ZET API 分发
│   │   ├── zer_libapi.cpp   #     ZER API 分发
│   │   ├── error_state.cpp/h#     线程级错误状态管理
│   │   ├── linux/           #     Linux 库初始化
│   │   └── windows/         #     Windows 库初始化
│   │
│   ├── loader/              #   加载器层（驱动管理与分发）
│   │   ├── ze_loader.cpp    #     主加载器（驱动发现、排序、初始化）
│   │   ├── ze_loader_api.cpp#     加载器公共 API 实现
│   │   ├── ze_loader_internal.h#   内部数据结构
│   │   ├── ze_ldrddi.cpp    #     加载器 DDI 拦截（ZE）
│   │   ├── zes_ldrddi.cpp   #     加载器 DDI 拦截（ZES）
│   │   ├── zet_ldrddi.cpp   #     加载器 DDI 拦截（ZET）
│   │   ├── zer_ldrddi.cpp   #     加载器 DDI 拦截（ZER）
│   │   ├── driver_discovery.h#    驱动发现接口
│   │   ├── ze_object.h      #     对象生命周期管理
│   │   ├── linux/           #     Linux 驱动发现
│   │   └── windows/         #     Windows 驱动发现
│   │
│   ├── layers/              #   可选层实现
│   │   ├── tracing/         #     追踪层（API 调用记录）
│   │   └── validation/      #     验证层（参数校验）
│   │
│   ├── drivers/             #   驱动实现
│   │   └── null/            #     空驱动（测试用模拟驱动）
│   │
│   └── utils/               #   工具函数
│       ├── logging.cpp/h    #     日志基础设施（spdlog）
│       └── ze_to_string.h   #     对象序列化工具
│
├── test/                    # 单元测试（GoogleTest）
├── samples/                 # 示例程序
│   └── zello_world/         #   Hello World 示例
├── bindings/                # 语言绑定
│   └── sysman/python/       #   Python 系统管理绑定（PyZES）
├── third_party/             # 第三方依赖
│   ├── spdlog_headers/      #   日志库
│   └── xla/                 #   XLA 支持
├── doc/                     # 文档
├── scripts/                 # 构建脚本
└── CMakeLists.txt           # 根构建配置
```

---

## 3. 整体软件架构

Level Zero 采用**分层拦截式架构**，核心分为四层：

```
┌────────────────────────────────────────────────────────────────────┐
│                        用户应用程序                                 │
│            zeInit() → zeDriverGet() → zeDeviceGet() → ...         │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                    库层 (source/lib/)                               │
│  libze_loader.so / ze_loader.dll                                   │
│                                                                    │
│  · 应用程序直接链接的共享库                                          │
│  · 管理全局 DDI 函数指针表（ze_dditable_t）                         │
│  · 原子操作保证线程安全                                              │
│  · std::call_once 保证单次初始化                                    │
└───────────────────────────────┬────────────────────────────────────┘
                                │ DDI 函数指针调用
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│               可选层链 (source/layers/)                             │
│                                                                    │
│  ┌──────────────────┐    ┌──────────────────┐                     │
│  │   追踪层          │ →  │   验证层          │                     │
│  │ (Tracing Layer)  │    │(Validation Layer)│                     │
│  │                  │    │                  │                     │
│  │ · 前置/后置回调   │    │ · 参数校验        │                     │
│  │ · API 日志记录    │    │ · 句柄生命周期    │                     │
│  │ · 性能分析       │    │ · 内存泄漏检测    │                     │
│  └──────────────────┘    └──────────────────┘                     │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                   加载器层 (source/loader/)                         │
│                                                                    │
│  · 驱动发现（平台相关：Linux/Windows）                               │
│  · 多驱动管理和分发                                                  │
│  · 驱动排序（按设备类型：独显 > 集显 > NPU）                         │
│  · 句柄翻译（多驱动场景下的句柄映射）                                 │
│  · DDI 表路由到正确的驱动实现                                        │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│                    驱动层 (Hardware Drivers)                        │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ Intel GPU    │  │ Intel NPU    │  │  Null Driver │             │
│  │ Driver       │  │ Driver       │  │  (测试用)    │             │
│  │              │  │              │  │              │             │
│  │libze_intel_  │  │libze_intel_  │  │ ze_null.so   │             │
│  │gpu.so.1     │  │npu.so.1     │  │              │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└────────────────────────────────────────────────────────────────────┘
```

**核心设计原则**：
1. **薄抽象层**：最小化 API 调用开销，接近硬件直接访问
2. **拦截式分发**：所有 API 调用通过 DDI 函数指针表分发，支持层插入
3. **多驱动透明**：应用无需感知底层存在多个驱动
4. **延迟加载**：多驱动场景下驱动按需加载，减少启动开销

---

## 4. 核心组件详解

### 4.1 API 头文件层 (include/)

API 头文件定义了 Level Zero 的完整公共接口，采用 **C 语言风格 API** 保证跨语言兼容性。

#### 句柄系统

所有 Level Zero 对象使用**不透明指针句柄**：

```c
typedef struct _ze_driver_handle_t *ze_driver_handle_t;     // 驱动实例
typedef struct _ze_device_handle_t *ze_device_handle_t;     // 计算设备
typedef struct _ze_context_handle_t *ze_context_handle_t;   // 执行上下文
typedef struct _ze_command_queue_handle_t *ze_command_queue_handle_t;  // 命令队列
typedef struct _ze_command_list_handle_t *ze_command_list_handle_t;    // 命令列表
typedef struct _ze_module_handle_t *ze_module_handle_t;     // 编译模块
typedef struct _ze_kernel_handle_t *ze_kernel_handle_t;     // 计算内核
typedef struct _ze_event_handle_t *ze_event_handle_t;       // 同步事件
```

#### 版本管理

```c
#define ZE_MAKE_VERSION(_major, _minor) ((_major << 16) | (_minor & 0x0000ffff))
#define ZE_MAJOR_VERSION(_ver) (_ver >> 16)
#define ZE_MINOR_VERSION(_ver) (_ver & 0x0000ffff)
// 当前版本：ZE_API_VERSION_CURRENT = ZE_MAKE_VERSION(1, 15)
```

#### 类型标记系统

每个结构体包含 `stype` 字段用于运行时类型识别，确保前向兼容：

```c
typedef enum _ze_structure_type_t {
    ZE_STRUCTURE_TYPE_DRIVER_PROPERTIES = 0x1,
    ZE_STRUCTURE_TYPE_DEVICE_PROPERTIES = 0x3,
    ZE_STRUCTURE_TYPE_COMMAND_QUEUE_DESC = 0xe,
    // 扩展类型从 0x10000 开始
    ZE_STRUCTURE_TYPE_PCI_EXT_PROPERTIES = 0x10008,
    // ...
} ze_structure_type_t;
```

#### DDI 句柄内嵌表

每个 Level Zero 句柄内部嵌入了指向所属驱动 DDI 表的指针：

```c
// ze_ddi_common.h
typedef struct _ze_handle_t {
    ze_dditable_driver_t  *pCore;     // ZE DDI 指针
    zet_dditable_driver_t *pTools;    // ZET DDI 指针
    zes_dditable_driver_t *pSysman;   // ZES DDI 指针
    zer_dditable_driver_t *pRuntime;  // ZER DDI 指针
} ze_handle_t;
```

这种设计使得后续 API 调用可以直接通过句柄找到对应驱动的函数，无需全局查找。

---

### 4.2 库层 (source/lib/)

库层是用户应用程序直接链接的共享库（`libze_loader.so`），负责 API 入口分发。

#### 核心类：`ze_lib::context_t`

```cpp
// ze_lib.h
class context_t {
    // DDI 表（原子指针保证线程安全读取）
    std::atomic<ze_dditable_t *>  zeDdiTable  = {nullptr};
    std::atomic<zet_dditable_t *> zetDdiTable = {nullptr};
    std::atomic<zes_dditable_t *> zesDdiTable = {nullptr};
    std::atomic<zer_dditable_t *> zerDdiTable = {nullptr};

    // 初始化控制
    std::once_flag initOnce;         // 保证 Init() 只执行一次
    std::once_flag initOnceDrivers;  // 保证 InitDrivers() 只执行一次
    bool isInitialized = false;

    // 核心方法
    ze_result_t Init(ze_init_flags_t flags, ...);       // 初始化加载器
    ze_result_t zeDdiTableInit(ze_api_version_t ver);   // 填充 ZE DDI 表
    ze_result_t zetDdiTableInit(ze_api_version_t ver);  // 填充 ZET DDI 表
    ze_result_t zesDdiTableInit(ze_api_version_t ver);  // 填充 ZES DDI 表
    ze_result_t zerDdiTableInit(ze_api_version_t ver);  // 填充 ZER DDI 表
};

extern context_t *context;  // 全局单例
```

#### API 分发模式

库层中每个公共 API 函数遵循相同的分发模式（以 `zeDeviceGet` 为例）：

```cpp
// ze_libapi.cpp
ze_result_t ZE_APICALL zeDeviceGet(
    ze_driver_handle_t hDriver,
    uint32_t* pCount,
    ze_device_handle_t* phDevices)
{
    // 1. 检查库是否正在析构
    if (ze_lib::destruction) {
        return ZE_RESULT_ERROR_UNINITIALIZED;
    }

    // 2. 从 DDI 表获取函数指针（原子加载）
    auto pfnGet = ze_lib::context->zeDdiTable.load()->Device.pfnGet;
    if (nullptr == pfnGet) {
        if (!ze_lib::context->isInitialized)
            return ZE_RESULT_ERROR_UNINITIALIZED;
        else
            return ZE_RESULT_ERROR_UNSUPPORTED_FEATURE;
    }

    // 3. 标记 ZE API 正在使用
    ze_lib::context->zeInuse = true;

    // 4. 通过函数指针分发到加载器/驱动
    return pfnGet(hDriver, pCount, phDevices);
}
```

#### 库初始化（Linux 平台）

```cpp
// source/lib/linux/lib_init.cpp
#ifndef L0_STATIC_LOADER_BUILD
void __attribute__((constructor)) createLibContext() {
    context = new context_t;  // dlopen 时自动创建
}
void __attribute__((destructor)) deleteLibContext() {
    delete context;           // dlclose 时自动销毁
}
#endif
```

#### DDI 表填充流程

```cpp
// ze_libddi.cpp
ze_result_t context_t::zeDdiTableInit(ze_api_version_t version) {
    // 从加载器获取各组件的 DDI 表
    auto getTable = reinterpret_cast<ze_pfnGetGlobalProcAddrTable_t>(
        GET_FUNCTION_PTR(loader, "zeGetGlobalProcAddrTable"));
    result = getTable(version, &initialzeDdiTable.Global);

    // 依次填充：Driver、Device、Context、CommandQueue...
    // 共 23+ 个 DDI 子表
    result = getDriverTable(version, &initialzeDdiTable.Driver);
    result = getDeviceTable(version, &initialzeDdiTable.Device);
    result = getContextTable(version, &initialzeDdiTable.Context);
    // ...

    // 将初始表设为活跃表（原子操作）
    zeDdiTable.store(&initialzeDdiTable);
}
```

---

### 4.3 加载器层 (source/loader/)

加载器是 Level Zero 的核心调度中枢，负责驱动发现、加载和多驱动管理。

#### 核心数据结构

```cpp
// ze_loader_internal.h

// 驱动实例
struct driver_t {
    HMODULE handle;                   // 动态库句柄
    ze_result_t initStatus;           // 初始化状态
    dditable_t dditable;              // 该驱动的全部 DDI 表
    std::string name;                 // 驱动名称/路径
    bool driverInuse;                 // 是否被使用
    zel_driver_type_t driverType;     // 驱动类型（GPU/NPU 等）
    bool pciOrderingRequested;        // 是否请求 PCI 排序
};

// 全局加载器上下文
class context_t {
    driver_vector_t allDrivers;       // 所有已发现的驱动
    driver_vector_t zeDrivers;        // 用于 ZE/ZET/ZES 的驱动
    driver_vector_t zesDrivers;       // 用于系统管理的驱动

    bool intercept_enabled;           // 是否启用拦截模式
    bool debugTraceEnabled;           // 是否启用调试跟踪
    bool tracingLayerEnabled;         // 追踪层状态
    ze_api_version_t version;         // 请求的 API 版本

    // 对象工厂（用于遗留拦截模式下的句柄包装）
    ze_driver_factory_t ze_driver_factory;
    ze_device_factory_t ze_device_factory;
    ze_context_factory_t ze_context_factory;
    // ... 30+ 种对象工厂

    // 核心方法
    ze_result_t init();
    ze_result_t init_driver(driver_t &driver, ...);
    bool driverSorting(driver_vector_t *drivers, ...);
    void driverOrdering(driver_vector_t *drivers);
};
```

#### 驱动发现机制

**Linux 平台** (`source/loader/linux/driver_discovery_lin.cpp`)：

```
搜索路径组装：
  1. LD_LIBRARY_PATH 环境变量
  2. 标准路径：/lib, /usr/lib, /usr/local/lib
  3. 多架构路径：/usr/lib/x86_64-linux-gnu 等
  4. 分发特定路径：/lib64, /usr/lib64
  5. /etc/ld.so.conf 及其包含的配置文件

已知驱动名称：
  · libze_intel_gpu.so.1        → GPU 驱动
  · libze_intel_gpu_legacy1.so.1 → 旧版 GPU 驱动
  · libze_intel_vpu.so.1        → VPU 驱动
  · libze_intel_npu.so.1        → NPU 驱动

自定义驱动：
  · 通过 ZE_ENABLE_ALT_DRIVERS 环境变量指定额外驱动路径
```

#### 驱动排序策略

加载器按设备类型对驱动排序，确保应用获得最优设备：

```
默认排序（独显优先）：
  1. ZEL_DRIVER_TYPE_DISCRETE_GPU    — 仅含独立 GPU 的驱动
  2. ZEL_DRIVER_TYPE_GPU             — 含独显和集显的驱动
  3. ZEL_DRIVER_TYPE_INTEGRATED_GPU  — 仅含集成 GPU 的驱动
  4. ZEL_DRIVER_TYPE_MIXED           — 混合设备（GPU + NPU）
  5. ZEL_DRIVER_TYPE_OTHER           — 其他设备类型
  6. ZEL_DRIVER_TYPE_NPU             — 仅含 NPU 的驱动

PCI 排序模式（ZE_ENABLE_PCI_ID_DEVICE_ORDER=1）：
  1. ZEL_DRIVER_TYPE_INTEGRATED_GPU  — 集成 GPU 优先
  2. ZEL_DRIVER_TYPE_GPU
  3. ZEL_DRIVER_TYPE_DISCRETE_GPU
  4. ZEL_DRIVER_TYPE_MIXED
  5. ZEL_DRIVER_TYPE_OTHER
  6. ZEL_DRIVER_TYPE_NPU
```

#### 对象管理模板

```cpp
// ze_object.h
template<typename _handle_t>
class object_t {
    ze_dditable_driver_t  *pCore;     // 核心 DDI
    zet_dditable_driver_t *pTools;    // 工具 DDI
    zes_dditable_driver_t *pSysman;   // 系统管理 DDI
    zer_dditable_driver_t *pRuntime;  // 运行时 DDI
    _handle_t handle;                 // 驱动原生句柄
    dditable_t* dditable;             // 完整 DDI 表
};
```

---

### 4.4 层系统 (source/layers/)

层系统实现了**堆叠式拦截器模式**，在 API 调用到达驱动之前进行处理。

#### 4.4.1 追踪层 (Tracing Layer)

**目的**：记录所有 API 调用、提供性能分析和调试能力。

**核心类**：

| 类 | 职责 |
|----|------|
| `APITracer` | 追踪器抽象基类 |
| `APITracerImp` | 具体实现，管理回调和状态转换 |
| `APITracerContextImp` | 全局上下文，管理所有活跃追踪器 |
| `ThreadPrivateTracerData` | 线程私有数据，持有活跃追踪器数组引用 |

**追踪器状态机**：

```
disabledState（禁用）
    ↓ setPrologues/setEpilogues（设置回调）
enabledState（启用，回调执行中）
    ↓ enableTracer(false)
disabledWaitingState（等待所有线程释放引用）
    ↓ 所有线程释放后
disabledState（可重新启用或销毁）
```

**API 拦截模式**（`ze_trcddi.cpp`）：

```cpp
__zedlllocal ze_result_t ZE_APICALL zeInit(ze_init_flags_t flags) {
    // 1. 获取下一层函数指针
    auto pfnInit = context.zeDdiTable.Global.pfnInit;

    // 2. 防止递归拦截
    ZE_HANDLE_TRACER_RECURSION(context.zeDdiTable.Global.pfnInit, flags);

    // 3. 捕获参数
    ze_init_params_t tracerParams = {&flags};

    // 4. 构建回调数据
    APITracerCallbackDataImp<ze_pfnInitCb_t> apiCallbackData;
    ZE_GEN_PER_API_CALLBACK_STATE(apiCallbackData, ze_pfnInitCb_t,
                                   Global, pfnInitCb);

    // 5. 通过包装器执行：前置回调 → 实际 API → 后置回调
    return APITracerWrapperImp(
        context.zeDdiTable.Global.pfnInit,  // 实际函数
        &tracerParams,                       // 参数
        apiCallbackData.prologCallbacks,     // 前置回调列表
        apiCallbackData.epilogCallbacks,     // 后置回调列表
        *tracerParams.pflags);               // 转发参数
}
```

**追踪包装器执行流程**：

```
前置回调(prologue) × N
    ↓
实际 API 调用
    ↓
后置回调(epilogue) × N
    ↓
返回结果
```

#### 4.4.2 验证层 (Validation Layer)

**目的**：在开发阶段检查 API 使用正确性。

**核心组件**：

```cpp
// ze_validation_layer.h
class context_t {
    bool enableHandleLifetime;        // 追踪句柄创建/销毁
    bool enableThreadingValidation;   // 检查线程安全
    bool verboseLogging;              // 详细日志

    std::vector<validationChecker *> validationHandlers;  // 验证检查器链
    std::unique_ptr<HandleLifetimeValidation> handleLifetime;
};
```

**验证入口点模式**（每个 API 都有对应的 Prologue 和 Epilogue）：

```cpp
class ZEValidationEntryPoints {
public:
    virtual ze_result_t zeInitPrologue(ze_init_flags_t flags)
        { return ZE_RESULT_SUCCESS; }
    virtual ze_result_t zeInitEpilogue(ze_init_flags_t flags, ze_result_t result)
        { return ZE_RESULT_SUCCESS; }
    // ... 450+ API 的 Prologue/Epilogue 对
};
```

**可用的验证检查器**：

| 检查器 | 功能 |
|--------|------|
| `parameter_validation` | 参数类型和范围校验 |
| `handle_lifetime_tracking` | 句柄创建/销毁配对验证 |
| `basic_leak` | 内存泄漏检测 |
| `events_checker` | 事件状态验证 |
| `system_resource_tracker` | 系统资源使用追踪 |
| `performance` | 性能分析 |

**验证拦截模式**（`ze_valddi.cpp`）：

```cpp
ze_result_t zeInit(ze_init_flags_t flags) {
    // 1. 执行所有验证器的 Prologue
    for (auto &checker : context.validationHandlers) {
        auto result = checker->zeValidation->zeInitPrologue(flags);
        if (result != ZE_RESULT_SUCCESS) return result;
    }

    // 2. 调用下一层
    auto result = context.zeDdiTable.Global.pfnInit(flags);

    // 3. 执行所有验证器的 Epilogue
    for (auto &checker : context.validationHandlers) {
        checker->zeValidation->zeInitEpilogue(flags, result);
    }

    // 4. 日志记录
    logAndPropagateResult_zeInit(result, flags);

    return result;
}
```

---

### 4.5 驱动层 (source/drivers/)

仓库包含一个**空驱动（Null Driver）** 用于测试。

#### Null Driver 设计

- **用途**：无需物理硬件即可测试加载器和层基础设施
- **实现**：用 `malloc`/`free` 模拟内存分配，用简单句柄模拟设备对象
- **多实例**：可构建多个变体（`ze_null_test1`、`ze_null_test2`、`ze_intel_gpu`、`ze_intel_npu`）用于多驱动测试
- **驱动 ID**：每个实例通过 `ZEL_NULL_DRIVER_ID` 编译定义区分

```cpp
// ze_null.cpp 关键初始化
context_t::context_t() {
    // 注册 DDI 函数为 lambda
    zeDdiTable.Global.pfnInit = [](ze_init_flags_t) { return ZE_RESULT_SUCCESS; };

    zeDdiTable.Device.pfnGet = [](ze_driver_handle_t, uint32_t* pCount,
                                   ze_device_handle_t* phDevices) {
        *pCount = 1;
        if (phDevices) phDevices[0] = reinterpret_cast<ze_device_handle_t>(
            0x80800000 >> ZEL_NULL_DRIVER_ID);  // 唯一句柄
        return ZE_RESULT_SUCCESS;
    };

    zeDdiTable.Memory.pfnAllocDevice = [](auto..., void** pptr, size_t size, ...) {
        *pptr = malloc(size);  // 用系统 malloc 模拟
        return ZE_RESULT_SUCCESS;
    };
}
```

---

### 4.6 工具层 (source/utils/)

#### 日志系统

基于 **spdlog** 实现，支持文件和控制台输出：

```cpp
// logging.h
class Logger {
public:
    void log_trace(const std::string &msg);
    void log_debug(const std::string &msg);
    void log_info(const std::string &msg);
    void log_warning(const std::string &msg);
    void log_error(const std::string &msg);
};
```

#### 对象序列化

为调试提供枚举值到字符串的转换：

```cpp
// ze_to_string.h
std::string to_string(ze_result_t value);       // "ZE_RESULT_SUCCESS"
std::string to_string(ze_device_type_t value);   // "ZE_DEVICE_TYPE_GPU"
std::string to_string(ze_init_flags_t value);    // "ZE_INIT_FLAG_GPU_ONLY"
```

---

## 5. 四大 API 子系统

Level Zero 包含四个协同工作的 API 子系统：

```
┌──────────────────────────────────────────────────────────────┐
│                     应用程序                                  │
├──────────┬──────────┬──────────┬──────────────────────────────┤
│  ZE API  │ ZES API  │ ZET API  │         ZER API             │
│ (核心)   │ (系统)   │ (工具)   │         (运行时)            │
├──────────┴──────────┴──────────┴──────────────────────────────┤
│                   ze_api.h（共享基础类型）                     │
├──────────────────────────────────────────────────────────────┤
│                   Loader / DDI 分发层                         │
└──────────────────────────────────────────────────────────────┘
```

### 5.1 ZE — 核心计算 API

| 方面 | 详情 |
|------|------|
| **头文件** | `ze_api.h`（18,715 行） |
| **功能** | GPU 编程的核心接口 |
| **主要模块** | 驱动管理、设备管理、上下文、命令队列/列表、内存管理、模块/内核、事件/同步、镜像/采样器 |
| **关键函数** | `zeInit`, `zeDriverGet`, `zeDeviceGet`, `zeContextCreate`, `zeCommandListCreate`, `zeModuleCreate`, `zeKernelCreate`, `zeMemAllocDevice` |

### 5.2 ZES — 系统管理 API

| 方面 | 详情 |
|------|------|
| **头文件** | `zes_api.h`（8,809 行） |
| **功能** | 设备级系统管理 |
| **管理对象** | 电源域（Power）、频率域（Frequency）、温度传感器（Temperature）、风扇（Fan）、引擎组（Engine）、内存模块（Memory）、固件（Firmware）、RAS 错误检测、待机控制（Standby）、超频（Overclock） |
| **句柄复用** | `zes_driver_handle_t = ze_driver_handle_t` |

### 5.3 ZET — 工具/调试 API

| 方面 | 详情 |
|------|------|
| **头文件** | `zet_api.h`（3,972 行） |
| **功能** | 性能分析、调试、指标收集 |
| **主要对象** | Metric Group/Handle、Metric Streamer、Metric Query Pool、Debug Session、API Tracer |
| **句柄复用** | Device、Context、CommandList 等直接复用 ZE 核心类型 |

### 5.4 ZER — 运行时辅助 API

| 方面 | 详情 |
|------|------|
| **头文件** | `zer_api.h`（162 行） |
| **功能** | 运行时便利工具 |
| **主要功能** | 默认描述符（`zeDefaultGPUImmediateCommandQueueDesc`）、错误描述获取（`zerGetLastErrorDescription`）、设备句柄转换 |

---

## 6. DDI (设备驱动接口) 机制

DDI 是 Level Zero 架构的核心调度机制，通过**函数指针表**实现层间解耦。

### DDI 表结构

```c
// ze_ddi.h

// 函数指针类型定义
typedef ze_result_t (ZE_APICALL *ze_pfnDeviceGet_t)(
    ze_driver_handle_t, uint32_t*, ze_device_handle_t*);

// 模块级 DDI 表
typedef struct _ze_device_dditable_t {
    ze_pfnDeviceGet_t                  pfnGet;
    ze_pfnDeviceGetProperties_t        pfnGetProperties;
    ze_pfnDeviceGetComputeProperties_t pfnGetComputeProperties;
    ze_pfnDeviceGetMemoryProperties_t  pfnGetMemoryProperties;
    // ... 更多函数指针
} ze_device_dditable_t;

// 完整 DDI 表（值语义，库层使用）
typedef struct _ze_dditable_t {
    ze_global_dditable_t    Global;
    ze_driver_dditable_t    Driver;
    ze_device_dditable_t    Device;
    ze_context_dditable_t   Context;
    ze_command_queue_dditable_t CommandQueue;
    ze_command_list_dditable_t  CommandList;
    // ... 共 23+ 子表
} ze_dditable_t;

// 完整 DDI 表（指针语义，驱动层使用）
typedef struct _ze_dditable_driver_t {
    ze_global_dditable_t     *Global;
    ze_driver_dditable_t     *Driver;
    ze_device_dditable_t     *Device;
    // ...
    ze_api_version_t version;
    bool isValidFlag;
} ze_dditable_driver_t;
```

### DDI 分发链

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────────┐
│ 库层 DDI 表     │ ──→ │ 追踪层 DDI 表    │ ──→ │ 验证层 DDI 表     │
│ (ze_dditable_t) │     │ (ze_dditable_t)  │     │ (ze_dditable_t)   │
│                 │     │                  │     │                   │
│ pfnDeviceGet ───┤     │ pfnDeviceGet ────┤     │ pfnDeviceGet ─────┤
│ 指向追踪层      │     │ 指向验证层       │     │ 指向加载器        │
└─────────────────┘     └──────────────────┘     └───────────────────┘
                                                           │
                                                           ▼
                                                 ┌───────────────────┐
                                                 │ 加载器 DDI 表     │
                                                 │                   │
                                                 │ pfnDeviceGet ─────┤
                                                 │ 指向驱动实现      │
                                                 └───────────────────┘
                                                           │
                                                           ▼
                                                 ┌───────────────────┐
                                                 │ 驱动 DDI 表       │
                                                 │ （硬件实现）      │
                                                 └───────────────────┘
```

### DDI 表初始化过程

```
1. 库层调用 zeGetGlobalProcAddrTable() 从加载器获取 Global DDI 表
2. 库层依次获取 Driver、Device、Context 等子表
3. 如果启用追踪层：追踪层的 DDI 表覆盖库层表，追踪层保存原始指针
4. 如果启用验证层：验证层的 DDI 表插入追踪层和加载器之间
5. 加载器从驱动获取驱动级 DDI 表
```

---

## 7. 完整调用流程分析

### 7.1 初始化流程 (zeInit)

```
用户代码: zeInit(ZE_INIT_FLAG_GPU_ONLY)
    │
    ▼
━━━ 库层 (ze_libapi.cpp) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  std::call_once(initOnce) {
    │      ze_lib::context->Init(flags)
    │  }
    │
    ├─▶ Init() — 核心初始化
    │   │
    │   ├─ 加载 libze_loader.so（静态构建模式）
    │   │  └─ dlopen("libze_loader.so")
    │   │
    │   ├─ 调用 zeLoaderInit()
    │   │
    │   ├─ 获取加载器版本 zelLoaderGetVersion()
    │   │
    │   ├─ DDI 表初始化链：
    │   │  ├─ zeDdiTableInit(version)    ← 核心 API
    │   │  ├─ zetDdiTableInit(version)   ← 工具 API
    │   │  ├─ zesDdiTableInit(version)   ← 系统管理 API
    │   │  ├─ zerDdiTableInit(version)   ← 运行时 API
    │   │  └─ zelTracingDdiTableInit()   ← 追踪 API
    │   │
    │   └─ 保存初始 DDI 表（供追踪层后续使用）
    │
    │  获取 pfnInit 函数指针
    │  调用 pfnInit(flags)
    │
    ▼
━━━ 加载器层 (ze_loader.cpp) ━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  zeLoaderInit():
    │  │
    │  ├─ 检查环境变量配置
    │  │  ├─ ZE_ENABLE_LOADER_DEBUG_TRACE
    │  │  ├─ ZE_ENABLE_PCI_ID_DEVICE_ORDER
    │  │  └─ ZE_ENABLE_NULL_DRIVER
    │  │
    │  ├─ discoverEnabledDrivers()
    │  │  └─ 搜索系统路径查找驱动库
    │  │     ├─ libze_intel_gpu.so.1
    │  │     ├─ libze_intel_npu.so.1
    │  │     └─ 自定义驱动 (ZE_ENABLE_ALT_DRIVERS)
    │  │
    │  ├─ 加载驱动策略：
    │  │  ├─ 单驱动：立即加载 → dlopen()
    │  │  └─ 多驱动：延迟加载（首次 API 调用时）
    │  │
    │  ├─ 加载可选层：
    │  │  ├─ 验证层 (ZE_ENABLE_VALIDATION_LAYER=1)
    │  │  └─ 追踪层 (始终加载，可选启用)
    │  │
    │  ├─ 决定拦截模式：
    │  │  └─ intercept_enabled = (多驱动 || ZE_ENABLE_LOADER_INTERCEPT)
    │  │
    │  └─ 返回 ZE_RESULT_SUCCESS
    │
    │  zeInit(flags):
    │  │
    │  ├─ 遍历所有驱动：
    │  │  ├─ 加载驱动库（如未加载）
    │  │  ├─ 初始化驱动 DDI 表
    │  │  ├─ 调用驱动的 zeInit(flags)
    │  │  └─ 记录成功/失败状态
    │  │
    │  └─ 至少一个驱动成功 → 返回 SUCCESS
    │
    ▼
━━━ 驱动层 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  driver->zeInit(flags)
    │  └─ 硬件初始化
    │
    ▼
返回 ZE_RESULT_SUCCESS
```

### 7.2 驱动发现与排序

排序发生在首次调用 `zeDriverGet()` 时：

```
zeDriverGet(pCount, phDrivers)
    │
    ▼
加载器拦截函数：
    │
    ├─ 加锁 sortMutex（防止竞态）
    │
    ├─ std::call_once → driverSorting()
    │  │
    │  ├─ 遍历每个驱动：
    │  │  ├─ 调用 zeDriverGet() 获取驱动句柄
    │  │  ├─ 调用 zeDeviceGet() 获取设备列表
    │  │  ├─ 查询每个设备类型（GPU/VPU/其他）
    │  │  └─ 对驱动分类：
    │  │     ├─ 仅独显 → DISCRETE_GPU
    │  │     ├─ 仅集显 → INTEGRATED_GPU
    │  │     ├─ 混合 GPU → GPU
    │  │     ├─ 仅 NPU → NPU
    │  │     ├─ GPU + NPU → MIXED
    │  │     └─ 其他 → OTHER
    │  │
    │  ├─ 按类型排序（升序或降序，取决于 PCI 模式）
    │  │
    │  └─ 应用 ZEL_DRIVERS_ORDER 自定义排序
    │
    ├─ 聚合所有驱动的句柄
    │
    └─ 返回总数或复制句柄到用户缓冲区
```

### 7.3 API 调用分发流程

以典型的 `zeCommandListCreate` 为例：

```
用户代码: zeCommandListCreate(hContext, hDevice, &desc, &hCommandList)
    │
    ▼
━━━ 库层 (ze_libapi.cpp) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  1. 检查 destruction 标志
    │  2. 原子加载 DDI 表：
    │     pfnCreate = zeDdiTable.load()->CommandList.pfnCreate
    │  3. 空指针检查（未初始化 vs 不支持）
    │  4. 调用 pfnCreate(hContext, hDevice, &desc, &hCommandList)
    │
    ▼
━━━ 追踪层 (ze_trcddi.cpp) — 如果启用 ━━━━━━━━━━━━━━━
    │
    │  1. 执行所有前置回调(prologue)
    │  2. 调用 context.zeDdiTable.CommandList.pfnCreate(...)
    │  3. 执行所有后置回调(epilogue)
    │  4. 返回结果
    │
    ▼
━━━ 验证层 (ze_valddi.cpp) — 如果启用 ━━━━━━━━━━━━━━━
    │
    │  1. 执行所有 Prologue 检查
    │     ├─ 参数非空检查
    │     ├─ 句柄有效性检查
    │     └─ 线程安全检查
    │  2. 调用 context.zeDdiTable.CommandList.pfnCreate(...)
    │  3. 执行所有 Epilogue 检查
    │     └─ 注册新句柄到生命周期追踪
    │  4. 日志记录结果
    │  5. 返回结果
    │
    ▼
━━━ 加载器层 (ze_ldrddi.cpp) ━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  拦截模式（多驱动）：
    │  1. 从 hContext 句柄提取驱动 DDI 表
    │  2. 路由到正确驱动：
    │     driver->dditable.ze.CommandList.pfnCreate(...)
    │  3. 包装返回的句柄（加入对象工厂）
    │
    │  直通模式（单驱动）：
    │  1. 直接调用驱动函数
    │
    ▼
━━━ 驱动层 (Intel GPU Driver) ━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  创建命令列表硬件资源
    │  返回句柄
    │
    ▼
返回 ze_command_list_handle_t 给用户
```

### 7.4 层拦截链

```
完整调用链（所有层启用）：

  用户 → 库层 → 追踪层 → 验证层 → 加载器 → 驱动

简化调用链（无层）：

  用户 → 库层 → 加载器 → 驱动

单驱动直通（无拦截）：

  用户 → 库层 → 驱动
```

---

## 8. 关键设计模式

### 8.1 拦截器/代理模式 (Interceptor/Proxy)

**应用位置**：层系统、加载器  
**实现方式**：DDI 函数指针表替换  
**效果**：透明地在 API 调用链中插入处理逻辑

### 8.2 工厂模式 (Factory)

**应用位置**：加载器 `context_t` 中的 30+ 对象工厂  
**实现方式**：`ze_driver_factory_t`、`ze_device_factory_t` 等  
**效果**：管理句柄包装和翻译

### 8.3 单例模式 (Singleton)

**应用位置**：`ze_lib::context`（全局库上下文）、`loader::context`（全局加载器上下文）  
**实现方式**：`extern context_t *context` + `std::call_once`  
**效果**：保证全局唯一初始化

### 8.4 延迟初始化 (Lazy Initialization)

**应用位置**：多驱动加载、驱动排序  
**实现方式**：多驱动场景下驱动在首次 API 调用时才加载  
**效果**：减少应用启动时间

### 8.5 观察者模式 (Observer)

**应用位置**：追踪层回调系统  
**实现方式**：前置/后置回调列表，多个追踪器可同时注册  
**效果**：解耦追踪逻辑与核心 API

### 8.6 策略模式 (Strategy)

**应用位置**：驱动排序  
**实现方式**：`driverSortComparator` 支持多种排序策略  
**效果**：可通过环境变量切换排序行为

### 8.7 模板方法模式 (Template Method)

**应用位置**：验证层入口点  
**实现方式**：`ZEValidationEntryPoints` 基类定义虚方法，子类覆盖  
**效果**：可插拔的验证检查器

### 8.8 RAII / GCC 构造/析构属性

**应用位置**：Linux 库初始化  
**实现方式**：`__attribute__((constructor))` / `__attribute__((destructor))`  
**效果**：库加载时自动初始化，卸载时自动清理

---

## 9. 构建系统

### 构建配置

| 配置项 | 值 |
|--------|-----|
| **构建工具** | CMake 3.12.0+ |
| **C++ 标准** | C++14 |
| **项目版本** | 1.28.2 |
| **测试框架** | GoogleTest v1.14.0（FetchContent） |
| **日志库** | spdlog（系统或内建） |

### 主要构建目标

```
ze_loader          — 动态加载器共享库
ze_loader_static   — 静态加载器库（L0_STATIC_LOADER_BUILD）
ze_tracing_layer   — 追踪层共享库
ze_validation_layer — 验证层共享库
ze_null            — 空驱动（测试用）
zello_world        — Hello World 示例
```

### 平台特定编译选项

**MSVC (Windows)**：
- `/DYNAMICBASE` — 地址空间随机化
- `/guard:cf` — 控制流保护
- `/Qspectre` — Spectre 漏洞缓解
- `/GL` — 全程序优化

**GCC/Clang (Linux)**：
- `-fPIC` — 位置无关代码
- `-fvisibility=hidden` — 默认隐藏符号
- `-Wall -Wnon-virtual-dtor` — 警告
- 可选 ASAN 支持

### 打包支持

| 格式 | 平台 |
|------|------|
| DEB | Debian/Ubuntu |
| RPM | Red Hat/SUSE |
| WIX (MSI) | Windows |
| ZIP | 通用 |

组件分离：`level-zero`（运行时）+ `level-zero-devel`（头文件和开发库）

---

## 10. 测试架构

### 测试文件

```
test/
├── driver_ordering_unit_tests.cpp         # 驱动排序逻辑测试
├── driver_ordering_helper_tests.cpp       # 排序辅助函数测试
├── driver_teardown_unit_tests.cpp         # 驱动清理测试
├── init_driver_unit_tests.cpp             # 驱动初始化测试
├── init_driver_dynamic_unit_tests.cpp     # 动态驱动加载测试
├── init_driver_unit_tests_common.h        # 共享测试工具
├── loader_api.cpp                         # 加载器 API 测试
├── loader_tracing_layer.cpp               # 追踪层测试
└── loader_validation_layer.cpp            # 验证层测试
```

### 测试范围

| 测试类别 | 覆盖内容 |
|----------|----------|
| **驱动生命周期** | 初始化、排序、清理 |
| **多驱动场景** | 驱动发现顺序、多驱动分发 |
| **加载器 API** | 版本查询、句柄翻译 |
| **追踪层** | 回调注册、前置/后置执行 |
| **验证层** | 参数校验、句柄追踪 |

### 空驱动测试变体

```
ze_null       — 主空驱动（ID=1）
ze_null_test1 — 测试驱动 1（ID=2）
ze_null_test2 — 测试驱动 2（ID=3）
ze_intel_gpu  — 模拟 GPU 驱动（ID=4）
ze_intel_npu  — 模拟 NPU 驱动（ID=4）
```

---

## 11. 线程安全与错误处理

### 线程安全机制

| 机制 | 位置 | 用途 |
|------|------|------|
| `std::atomic<ze_dditable_t*>` | 库层 | DDI 表读取无锁 |
| `std::call_once` | 库层 | 保证单次初始化 |
| `std::mutex sortMutex` | 加载器 | 保护驱动排序 |
| `std::atomic<> activeTracerArray` | 追踪层 | 活跃追踪器数组的无锁发布 |
| `ThreadPrivateTracerData` | 追踪层 | 线程私有的追踪器引用 |
| `std::mutex errorDescsMutex` | 错误管理 | 保护线程级错误描述 |

### 错误处理策略

```cpp
// 1. 驱动加载失败 — 非致命，继续加载下一个驱动
for (auto &driver : drivers) {
    result = init_driver(driver, flags);
    // 失败则记录日志，不中断
}

// 2. API 调用失败 — 返回错误码
if (nullptr == pfnCreate) {
    return ZE_RESULT_ERROR_UNINITIALIZED;  // 未初始化
    // 或 ZE_RESULT_ERROR_UNSUPPORTED_FEATURE;  // 不支持
}

// 3. 线程级错误描述
error_state::setErrorDesc("具体错误信息");
// 其他线程可通过 zerGetLastErrorDescription() 获取
```

### 错误状态管理

```cpp
// error_state.h/cpp
namespace error_state {
    std::unordered_map<std::thread::id, std::string> errorDescs;
    std::mutex errorDescsMutex;

    void setErrorDesc(const std::string &desc);   // 设置当前线程错误
    void getErrorDesc(const char **ppString);      // 获取当前线程错误
}
```

---

## 12. 环境变量配置

| 环境变量 | 默认值 | 功能 |
|----------|--------|------|
| **核心控制** | | |
| `ZE_ENABLE_LOADER_DEBUG_TRACE` | 0 | 启用加载器内部调试输出到 stderr |
| `ZE_ENABLE_LOADER_INTERCEPT` | 0 | 强制启用拦截模式 |
| `ZE_ENABLE_LOADER_DRIVER_DDI_PATH` | 1 | 启用 DDI 句柄扩展支持 |
| **驱动管理** | | |
| `ZE_ENABLE_NULL_DRIVER` | 0 | 加载空驱动（测试用） |
| `ZE_ENABLE_ALT_DRIVERS` | (空) | 自定义驱动路径（逗号分隔） |
| `ZE_ENABLE_PCI_ID_DEVICE_ORDER` | 0 | 使用 PCI 排序代替类型排序 |
| `ZEL_DRIVERS_ORDER` | (空) | 自定义驱动顺序 |
| **层控制** | | |
| `ZE_ENABLE_VALIDATION_LAYER` | 0 | 启用验证层 |
| `ZE_ENABLE_TRACING_LAYER` | 0 | 启用追踪层 |
| `ZE_ENABLE_HANDLE_LIFETIME` | 0 | 启用句柄生命周期追踪 |
| `ZE_ENABLE_THREADING_VALIDATION` | 0 | 启用线程安全检查 |
| **日志控制** | | |
| `ZEL_ENABLE_LOADER_LOGGING` | 0 | 启用日志文件输出 |
| `ZEL_LOADER_LOG_DIR` | ~/.oneapi_logs | 日志文件目录 |
| `ZEL_LOADER_LOGGING_LEVEL` | warn | 日志级别 (trace/debug/info/warn/error) |
| `ZEL_LOADER_LOG_CONSOLE` | 0 | 日志输出到 stderr |
| `ZEL_LOADER_LOG_PATTERN` | 时间戳+线程+级别 | 自定义日志格式 |
| `ZEL_LOADER_LOGGING_ENABLE_SUCCESS_PRINT` | 0 | 记录成功的 API 调用 |
| **调试/分析** | | |
| `ZET_ENABLE_PROGRAM_INSTRUMENTATION` | 0 | 启用程序插桩 |

---

## 13. 示例程序分析

### zello_world — Hello World 示例

**文件**：`samples/zello_world/zello_world.cpp`

**完整执行流程**：

```
1. 命令行参数解析
   ├─ -null   → 使用空驱动
   ├─ -ldr    → 强制拦截模式
   ├─ -val    → 启用验证层
   ├─ -trace  → 启用追踪层
   ├─ -npu    → 搜索 NPU 设备
   └─ -legacy_init → 使用旧版初始化

2. 初始化
   ├─ 设置环境变量（根据命令行参数）
   ├─ zeInit(0) 或 zeInitDrivers()
   └─ 查询加载器版本 zelLoaderGetVersions()

3. 设备发现
   ├─ zeDriverGet() → 获取所有驱动
   ├─ 遍历每个驱动：
   │  ├─ zeDeviceGet() → 获取设备列表
   │  └─ findDevice() → 查找匹配类型的设备
   └─ 收集所有有效的 (驱动, 设备) 对

4. 计算执行（每个有效设备）
   ├─ zeContextCreate()                    → 创建上下文
   ├─ zeCommandListCreateImmediate()       → 创建即时命令列表
   ├─ zeEventPoolCreate()                  → 创建事件池
   ├─ zeEventCreate()                      → 创建事件
   ├─ zeCommandListAppendSignalEvent()     → 设备发信号
   ├─ zeEventHostSynchronize(UINT64_MAX)   → 主机等待
   └─ 打印成功消息

5. 清理
   ├─ zeEventDestroy()
   ├─ zeEventPoolDestroy()
   ├─ zeCommandListDestroy()
   └─ zeContextDestroy()

6. 可选追踪演示
   ├─ zelEnableTracingLayer()    → 运行时启用
   ├─ zelTracerCreate()          → 创建追踪器
   └─ zelDisableTracingLayer()   → 禁用
```

---

## 14. 总结

### 架构特点

1. **分层设计**：清晰的四层架构（库层 → 层系统 → 加载器 → 驱动），每层职责明确
2. **DDI 分发机制**：通过函数指针表实现零开销的层间调用，避免虚函数开销
3. **多驱动透明**：应用程序无需感知底层驱动数量，加载器自动管理
4. **可插拔层系统**：追踪和验证层可通过环境变量动态启用/禁用
5. **跨平台支持**：通过平台抽象层（驱动发现、库初始化）支持 Linux 和 Windows
6. **线程安全**：原子操作、单次初始化、互斥锁等多重机制保证并发安全

### 工作流程概要

```
应用启动 → zeInit() → 驱动发现 → DDI 表建立 → 层链组装
    ↓
zeDriverGet() → 驱动排序 → 返回有序驱动句柄
    ↓
zeDeviceGet() → 返回设备句柄
    ↓
zeContextCreate() → zeCommandListCreate() → ...
    ↓
API 调用通过 DDI 表分发：库 → [追踪] → [验证] → 加载器 → 驱动
    ↓
结果沿调用链反向传播
```

### 关键设计决策

| 决策 | 原因 |
|------|------|
| C 语言 API | 最大化跨语言兼容性 |
| 不透明句柄 | 隐藏实现细节，支持句柄翻译 |
| DDI 函数指针表 | 零虚函数开销，支持层拦截 |
| 延迟驱动加载 | 减少多驱动场景启动时间 |
| 环境变量配置 | 无需重编译即可调整行为 |
| 标记类型系统 | 前向兼容，新结构不破坏旧代码 |
| 空驱动 | 无硬件依赖的测试基础设施 |
