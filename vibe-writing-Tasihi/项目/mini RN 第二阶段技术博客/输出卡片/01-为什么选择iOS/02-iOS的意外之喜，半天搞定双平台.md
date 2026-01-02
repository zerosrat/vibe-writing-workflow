## iOS 的意外之喜，半天搞定双平台

### 一个大胆的尝试

在 Android 的泥潭里挣扎了两天后，我几乎是自暴自弃地想："要不试试 iOS？反正都是 Apple 的东西，应该差不多吧。"

没想到，这个"差不多"成了整个项目的转折点。

### 第一个惊喜：JavaScriptCore 就在那里

打开 Xcode 文档，我简直不敢相信自己的眼睛——JavaScriptCore.framework 就在那里，像老朋友一样等着你。

不用编译，不用配置，不用版本匹配。就一句话：
```objective-c
#import <JavaScriptCore/JavaScriptCore.h>
```

搞定！

### 第二个惊喜：API 几乎一模一样

我抱着怀疑的态度，把 macOS 的 DeviceInfo 代码拷贝到 iOS 项目中。除了一个头文件引用（IOKit → UIKit），代码竟然直接编译通过了！

这意味着什么？
- JSCExecutor：100% 复用
- Bridge 通信逻辑：100% 复用
- 模块注册机制：100% 复用

### 第三个惊喜：CMake 原来这么智能

我原本以为要为 iOS 写一套全新的构建配置。结果呢？就加几行：

```cmake
if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
    set(PLATFORM_SOURCES src/ios/modules/deviceinfo/DeviceInfoModule.mm)
    find_library(UIKIT_FRAMEWORK UIKit)
else()
    # macOS 的配置
endif()
```

CMake 自动处理了剩下的一切。

### 半天完成的事实

上午 10 点开始，下午 3 点完成：
- 构建系统配置：1 小时
- DeviceInfo 模块适配：30 分钟
- 测试验证：1 小时
- 写文档：30 分钟

总计：5 小时

对比 Android 的 3 天预估，这不是差距，这是鸿沟。

### 生态系统的力量

这时候我懂了：为什么大公司都讲究"生态"。

Apple 的生态系统里，一切都是设计好的：
- macOS 和 iOS 共享核心框架
- 开发工具链高度集成
- API 设计保持一致性

这不是巧合，这是刻意的设计。

### 最深的启发

iOS 的成功适配告诉我：有时候，最好的选择不是最难的，而是最"顺"的。

顺着生态的流，而不是逆流而上。

> 这让我思考：技术的价值到底在于解决复杂问题，还是找到简单路径？