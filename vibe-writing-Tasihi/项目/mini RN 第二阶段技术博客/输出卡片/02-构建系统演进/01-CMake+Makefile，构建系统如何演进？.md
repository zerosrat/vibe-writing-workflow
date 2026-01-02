# CMake + Makefile，构建系统如何演进？

---

## 背景知识：Make 和 CMake 是什么？

### Make：最古老的构建工具

**Make 是什么？**

Make 是一个诞生于 1976 年的构建工具，它的工作原理很简单：描述文件之间的依赖关系，然后只重新编译"变化过的"文件。

---

## Make 工作原理：增量编译

### 问题场景：没有 Make 会怎样？

想象你有 5 个 C++ 源文件，每次修改一个文件都要重新编译所有 5 个文件：

```bash
# 没有构建工具时，每次都要这么干
echo "编译 main.cpp..."
g++ -c main.cpp

echo "编译 utils.cpp..."
g++ -c utils.cpp

echo "编译 network.cpp..."
g++ -c network.cpp

echo "编译 message.cpp..."
g++ -c message.cpp

echo "编译 logger.cpp..."
g++ -c logger.cpp

echo "链接所有 .o 文件..."
g++ -o myapp main.o utils.o network.o message.o logger.o

echo "编译完成！"
```

修改一行代码，编译需要 10 秒。一天改 50 次，浪费 500 秒（8 分钟）。

### Make 怎么解决增量编译？

Make 通过比较文件的"修改时间"来判断哪些文件需要重新编译。

```makefile
# Makefile：描述依赖关系
myapp: main.o utils.o network.o message.o logger.o
    @echo "🔗 链接可执行文件..."
    g++ -o myapp main.o utils.o network.o message.o logger.o

main.o: main.cpp utils.h message.h
    @echo "📝 编译 main.cpp..."
    g++ -c main.cpp

utils.o: utils.cpp utils.h
    @echo "📝 编译 utils.cpp..."
    g++ -c utils.cpp

network.o: network.cpp network.h utils.h
    @echo "📝 编译 network.cpp..."
    g++ -c network.cpp

message.o: message.cpp message.h
    @echo "📝 编译 message.cpp..."
    g++ -c message.cpp

logger.o: logger.cpp logger.h
    @echo "📝 编译 logger.cpp..."
    g++ -c logger.cpp
```

### 增量编译的实际演示

基于 Mini React Native 项目的真实构建过程：

**场景 1：首次完整构建**

```bash
$ make build
📦 Building JavaScript bundle...
🔧 Configuring build system...
🔨 Building Mini React Native...
[ 16%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/bridge/JSCExecutor.cpp.o
[ 33%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/modules/ModuleRegistry.cpp.o
[ 50%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/modules/NativeModule.cpp.o
[ 66%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/utils/JSONParser.cpp.o
[ 83%] Building CXX object CMakeFiles/mini_react_native.dir/src/macos/modules/deviceinfo/DeviceInfoModule.mm.o
[100%] Linking CXX static library libmini_react_native.a
[100%] Built target mini_react_native
✅ Build complete
```

**场景 2：修改单个源文件后的增量编译**

```bash
# 修改 JSCExecutor.cpp
$ touch src/common/bridge/JSCExecutor.cpp

# 再次构建 - 只重编译修改的文件
$ make build
🔨 Building Mini React Native...
[ 16%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/bridge/JSCExecutor.cpp.o
[100%] Linking CXX static library libmini_react_native.a
✅ Build complete
```

**Make 的判断过程**：

```
JSCExecutor.cpp 修改时间: 2025-01-02 14:30:00  ← 修改了
JSCExecutor.o   修改时间: 2025-01-02 14:20:00  ← 源文件比目标文件新，重新编译
ModuleRegistry.cpp 修改时间: 2025-01-02 14:00:00  ← 没修改
ModuleRegistry.o   修改时间: 2025-01-02 14:20:00  ← 目标文件更新，跳过
...
```

**结论**：只编译 `JSCExecutor.cpp`，其他 5 个文件不动。编译时间从 30 秒降到 5 秒。

---

### 依赖管理：自动处理复杂的依赖链

#### 场景：修改头文件会怎样？

```cpp
// utils.h
#ifndef UTILS_H
#define UTILS_H
std::string toLower(const std::string& s);
int add(int a, int b);  // ← 新增这个函数
#endif
```

**修改 `utils.h` 后，Make 的判断过程**：

```
utils.h  修改时间: 2025-01-02 14:35:00  ← 修改了

main.o: main.cpp utils.h message.h
    ↑ utils.h 是 main.o 的依赖，需要重新编译 main.cpp

utils.o: utils.cpp utils.h
    ↑ utils.h 是 utils.o 的依赖，需要重新编译 utils.cpp

network.o: network.cpp network.h utils.h
    ↑ utils.h 是 network.o 的依赖，需要重新编译 network.cpp
```

**运行 make**：

```bash
$ make
📝 编译 main.cpp...
📝 编译 utils.cpp...
📝 编译 network.cpp...
🔗 链接可执行文件...
```

**关键点**：Make 自动追踪依赖链，修改头文件后，所有依赖它的源文件都会重新编译。

---

### 依赖管理的真实场景

**项目结构**：

```
src/
├── main.cpp          # 入口文件
├── utils/
│   ├── utils.cpp
│   └── utils.h
├── network/
│   ├── client.cpp
│   ├── client.h
│   └── protocol.h
└── message/
    ├── builder.cpp
    └── builder.h
```

**对应的 Makefile**：

```makefile
# 可执行文件
myapp: main.o client.o builder.o utils.o
    @echo "🔗 链接 myapp..."
    g++ -o myapp main.o client.o builder.o utils.o

# 入口文件
main.o: main.cpp network/client.h message/builder.h utils/utils.h
    g++ -c main.cpp

# 网络模块
client.o: network/client.cpp network/client.h network/protocol.h utils/utils.h
    g++ -c network/client.cpp

# 消息模块
builder.o: message/builder.cpp message/builder.h utils/utils.h
    g++ -c message/builder.cpp

# 工具模块
utils.o: utils/utils.cpp utils/utils.h
    g++ -c utils/utils.cpp

# 清理目标
.PHONY: clean
clean:
    rm -f *.o myapp
```

**修改 `network/protocol.h` 后**：

```bash
$ make
📝 编译 network/client.cpp...   # 只编译依赖 protocol.h 的文件
🔗 链接 myapp...
```

---

## 任务编排：多步骤构建流程

### Mini React Native 项目的真实任务编排

基于实际项目代码，展示 Makefile 如何编排复杂的构建流程：

```makefile
# ============================================
# 变量定义 (Makefile 第 10-13 行)
# ============================================
BUILD_DIR = build
CMAKE_BUILD_TYPE ?= Debug
CORES = $(shell sysctl -n hw.ncpu)  # 动态检测 CPU 核心数

# ============================================
# 主要构建目标 - 依赖链设计
# ============================================

# 默认目标：make 等价于 make build
.PHONY: all
all: build

# 核心构建流程：js-build → configure → 实际编译
.PHONY: build
build: js-build configure
	@echo "🔨 Building Mini React Native..."
	@cd $(BUILD_DIR) && make -j$(CORES)
	@echo "✅ Build complete"

# ============================================
# 步骤 1：JavaScript 构建 (第 29-33 行)
# ============================================
.PHONY: js-build
js-build:
	@echo "📦 Building JavaScript bundle..."
	@npm run build    # 执行 rollup -c，生成 dist/bundle.js
	@echo "✅ JavaScript bundle built"

# ============================================
# 步骤 2：CMake 配置 (第 22-26 行)
# ============================================
.PHONY: configure
configure:
	@echo "🔧 Configuring build system..."
	@mkdir -p $(BUILD_DIR)
	@cd $(BUILD_DIR) && cmake -DCMAKE_BUILD_TYPE=$(CMAKE_BUILD_TYPE) -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
	@echo "✅ Configuration complete"

# ============================================
# 步骤 3：macos 构建
# ============================================
.PHONY: build
build: js-build configure
	@echo "🔨 Building Mini React Native..."
	@cd $(BUILD_DIR) && make -j$(CORES)
	@echo "✅ Build complete"

# ============================================
# 步骤 4：分层测试流程 (第 91-130 行)
# ============================================

# 完整测试：build → 4 个测试依次执行
.PHONY: test
test: build
	@echo "🧪 Running all tests..."
	@echo "\n📝 Test 1: Basic functionality test"
	@./$(BUILD_DIR)/mini_rn_test
	@echo "\n📝 Test 2: Module framework test"
	@./$(BUILD_DIR)/test_module_framework
	@echo "\n📝 Test 3: Integration test"
	@./$(BUILD_DIR)/test_integration
	@echo "\n📝 Test 4: Performance test"
	@./$(BUILD_DIR)/test_performance
	@echo "\n✅ All tests complete"

# 单独的测试目标 - 允许细粒度测试
.PHONY: test-basic
test-basic: build
	@echo "🧪 Running basic functionality test..."
	@./$(BUILD_DIR)/mini_rn_test

.PHONY: test-performance
test-performance: build
	@echo "🧪 Running performance test..."
	@./$(BUILD_DIR)/test_performance
```

### 任务编排的执行流程

基于 Mini React Native 项目的真实执行示例：

**完整构建流程**：

```bash
$ make build
📦 Building JavaScript bundle...
✅ JavaScript bundle built
🔧 Configuring build system...
✅ Configuration complete
🔨 Building Mini React Native...
[ 16%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/bridge/JSCExecutor.cpp.o
[ 33%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/modules/ModuleRegistry.cpp.o
[ 50%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/modules/NativeModule.cpp.o
[ 66%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/utils/JSONParser.cpp.o
[ 83%] Building CXX object CMakeFiles/mini_react_native.dir/src/macos/modules/deviceinfo/DeviceInfoModule.mm.o
[100%] Linking CXX static library libmini_react_native.a
[100%] Built target mini_react_native
[100%] Built target mini_rn_test
[100%] Built target test_module_framework
[100%] Built target test_integration
[100%] Built target test_performance
✅ Build complete
```

**完整测试流程**：

```bash
$ make test
🧪 Running all tests...

📝 Test 1: Basic functionality test
✅ JSCExecutor initialization successful
✅ Module registration successful
✅ Basic functionality test complete

📝 Test 2: Module framework test
✅ DeviceInfo module loaded
✅ Module framework test complete

📝 Test 3: Integration test
✅ JavaScript bundle loaded: dist/bundle.js
✅ Bridge communication test passed
✅ Integration test complete

📝 Test 4: Performance test
✅ Performance benchmark completed
✅ Performance test complete

✅ All tests complete
```

**iOS 构建流程**：

```bash
$ make ios-build
📦 Building JavaScript bundle...
✅ JavaScript bundle built
🔧 Configuring iOS build system...
✅ iOS configuration complete
🔨 Building Mini React Native for iOS...
[100%] Built target mini_react_native
[100%] Built target test_integration
✅ iOS build complete
```

**并行构建效果**：

```bash
# 使用 8 核 CPU 并行编译
$ make build
🔨 Building Mini React Native...
[ 16%] Building CXX object (并行编译 4 个文件)
[ 83%] Building CXX object
[100%] Linking CXX static library
```

**增量构建效果**：

```bash
# 修改单个文件后
$ touch src/common/bridge/JSCExecutor.cpp
$ make test
🔨 Building Mini React Native...
[ 16%] Building CXX object CMakeFiles/mini_react_native.dir/src/common/bridge/JSCExecutor.cpp.o
[100%] Linking CXX static library libmini_react_native.a
🧪 Running all tests...
...
```

**关键点**：
1. **依赖链自动执行**：`make test` 自动触发 `build`，`build` 自动触发 `js-build` 和 `configure`
2. **跳过未变化步骤**：JavaScript bundle 未变化时跳过 npm 构建
3. **并行编译优化**：自动使用系统 CPU 核心数进行并行编译
4. **平台隔离**：macOS 和 iOS 构建互不干扰

---

## Makefile 的核心机制深度解析

### 1. 增量编译机制的实现原理

**基于时间戳的依赖检查**：

```makefile
# Mini React Native 项目中的依赖链
build: js-build configure
	@cd $(BUILD_DIR) && make -j$(CORES)

js-build:
	@npm run build

configure:
	@cd $(BUILD_DIR) && cmake ..
```

**Make 的判断逻辑**：

```
1. 检查 build 目标的依赖：js-build, configure
2. 检查 dist/bundle.js 是否比 src/js/**/*.ts 新
   - 如果 src/js/ 有文件更新 → 执行 js-build
   - 否则跳过 js-build
3. 检查 build/Makefile 是否比 CMakeLists.txt 新
   - 如果 CMakeLists.txt 更新 → 执行 configure
   - 否则跳过 configure
4. 最后执行 C++ 编译（由 CMake 生成的 Makefile 处理增量编译）
```

**实际效果演示**：

```bash
# 场景 1：只修改 JavaScript
$ touch src/js/MessageQueue.js
$ make build
📦 Building JavaScript bundle...  ← 只执行 JS 构建
🔨 Building Mini React Native...  ← 跳过 CMake 配置
[100%] Built target mini_react_native

# 场景 2：只修改 CMake 配置
$ touch CMakeLists.txt
$ make build
🔧 Configuring build system...    ← 只执行 CMake 配置
🔨 Building Mini React Native...
[100%] Built target mini_react_native

# 场景 3：只修改 C++ 源码
$ touch src/common/bridge/JSCExecutor.cpp
$ make build
🔨 Building Mini React Native...  ← 跳过前两步，直接编译
[ 16%] Building CXX object ...JSCExecutor.cpp.o
[100%] Linking CXX static library libmini_react_native.a
```

### 2. 依赖管理的多层实现

**Makefile 层面的依赖**：

```makefile
# 显式依赖声明
test: build                        # test 依赖 build
build: js-build configure          # build 依赖 js-build 和 configure
ios-build: js-build ios-configure  # iOS 构建依赖 JS 构建和 iOS 配置
clean: js-clean                    # 清理依赖 JS 清理
```

**CMake 层面的依赖**（从 CMakeLists.txt）：

```cmake
# 库依赖
add_library(mini_react_native STATIC ${ALL_SOURCES})

# 可执行文件依赖
add_executable(mini_rn_test examples/test_basic.cpp)
target_link_libraries(mini_rn_test mini_react_native)  # 测试依赖库

add_executable(test_integration examples/test_integration.cpp)
target_link_libraries(test_integration mini_react_native)  # 集成测试依赖库
```

**平台条件依赖**：

```cmake
# 根据平台动态选择源文件
if(APPLE)
    if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
        set(PLATFORM_SOURCES src/ios/modules/deviceinfo/DeviceInfoModule.mm)
        find_library(UIKIT_FRAMEWORK UIKit)
        set(PLATFORM_FRAMEWORKS ${UIKIT_FRAMEWORK})
    else()
        set(PLATFORM_SOURCES src/macos/modules/deviceinfo/DeviceInfoModule.mm)
        find_library(IOKIT_FRAMEWORK IOKit)
        set(PLATFORM_FRAMEWORKS ${IOKIT_FRAMEWORK})
    endif()
endif()
```

**依赖图可视化**：

```
用户命令: make test
    ↓
test: build
    ↓
build: js-build configure
        ↓             ↓
    js-build          configure
        ↓                   ↓
    npm run build       cmake ..
        ↓                   ↓
    dist/bundle.js      build/Makefile
                            ↓
                        make -j8 (CMake 管理的依赖)
                            ↓
                        libmini_react_native.a
                            ↓
                        mini_rn_test (等 4 个可执行文件)
```

### 3. 任务编排的高级技巧

**并行执行优化**：

```makefile
# 动态检测 CPU 核心数
CORES = $(shell sysctl -n hw.ncpu)

# 传递给 CMake 进行并行编译
build: js-build configure
	@cd $(BUILD_DIR) && make -j$(CORES)
```

**条件执行**：

```makefile
# 开发模式：检查工具是否存在
dev:
	@if command -v fswatch &> /dev/null; then \
		echo "👀 Watching for file changes..."; \
		fswatch -o src examples CMakeLists.txt | while read; do \
			make build; \
		done; \
	else \
		echo "fswatch not found. Install with: brew install fswatch"; \
	fi
```

**平台特定任务**：

```makefile
# iOS 测试只在 iOS 构建后执行
ios-test: ios-build
	@./test_ios.sh all

# macOS 测试在通用构建后执行
test: build
	@./$(BUILD_DIR)/mini_rn_test
```

### 4. .PHONY 目标的重要性

在 Mini React Native 项目中，几乎所有目标都是 `.PHONY`：

```makefile
.PHONY: all build configure js-build js-clean test test-basic test-module
.PHONY: test-integration test-performance clean rebuild ios-build ios-configure
.PHONY: ios-test ios-test-deviceinfo install-deps format info dev help
```

**为什么需要 .PHONY？**

```bash
# 假设项目根目录有一个叫 "test" 的文件
$ touch test
$ make test
make: 'test' is up to date.  # 错误：Make 认为 test 文件已存在，跳过执行

# 使用 .PHONY 后
.PHONY: test
$ make test
🧪 Running all tests...     # 正确：强制执行，忽略同名文件
```

### 5. 变量和函数的巧妙运用

**动态变量**：

```makefile
# 根据系统动态设置
CORES = $(shell sysctl -n hw.ncpu)                    # macOS 获取 CPU 核心数
BUILD_DIR = build                                     # 构建目录
CMAKE_BUILD_TYPE ?= Debug                             # 默认 Debug，可通过环境变量覆盖
```

**条件变量**：

```makefile
# iOS 构建使用不同的构建目录
ios-configure:
	@mkdir -p $(BUILD_DIR)_ios  # 变量拼接
	@cd $(BUILD_DIR)_ios && cmake \
		-DCMAKE_SYSTEM_NAME=iOS \
		-DCMAKE_OSX_ARCHITECTURES=$$(uname -m) \  # Shell 命令替换
		..
```

**环境变量集成**：

```makefile
# 支持用户自定义构建类型
$ CMAKE_BUILD_TYPE=Release make build
# Makefile 中：CMAKE_BUILD_TYPE ?= Debug  # 如果环境变量未设置，使用 Debug
```

这种多层次的依赖管理和任务编排，让 Mini React Native 项目能够：
- **精确的增量编译**：只重新编译真正需要的部分
- **灵活的平台支持**：同一套脚本支持 macOS 和 iOS
- **高效的并行构建**：充分利用多核 CPU
- **用户友好的接口**：复杂的构建命令被简化为语义化的目标

---

## 适用场景

**适合使用 Make 的场景**：

1. **单平台项目**（只在一种操作系统上构建）
2. **需要精细控制编译步骤**（自定义编译选项、优化级别）
3. **增量编译能显著节省时间**（大型项目，频繁修改）
4. **需要多步骤构建流程**（代码生成 → 编译 → 测试 → 打包）

**不适合使用 Make 的场景**：

1. **跨平台项目**（不同操作系统需要不同的编译命令）
2. **复杂的项目结构**（大量依赖、多种配置）
3. **需要 IDE 集成**（Visual Studio、Xcode 等）

---

### CMake：跨平台的构建生成器

**CMake 是什么？**

CMake 不是"构建工具"，而是"构建系统的构建系统"。它读取 `CMakeLists.txt`，然后生成原生的构建文件。

```text
CMakeLists.txt
    ↓ CMake
生成的构建文件
    ├─ Windows: Visual Studio solution
    ├─ macOS: Xcode project
    ├─ Linux: Makefile
    └─ iOS: Xcode project + iOS SDK 配置
```

**问题场景：为什么需要 CMake？**

同一个 C++ 项目，在不同平台上编译命令完全不同：

```bash
# macOS
clang++ -o myapp main.cpp -std=c++17 -framework Foundation

# Linux
g++ -o myapp main.cpp -std=c++17 -lpthread

# Windows (Visual Studio)
cl.exe /Fe:myapp main.cpp /std:c++17
```

没有跨平台构建工具，需要维护三套脚本。

**CMake 怎么解决？**

```cmake
# CMakeLists.txt：平台无关的配置
project(MyApp)
set(CMAKE_CXX_STANDARD 17)

add_executable(myapp main.cpp)

# 根据平台链接不同的库
if(APPLE)
    target_link_libraries(myapp "-framework Foundation")
elseif(UNIX AND NOT APPLE)
    target_link_libraries(myapp pthread)
endif()
```

一套配置文件，所有平台都能用。

**CMake 的核心价值**：

1. **跨平台**：同一份配置支持 Windows、macOS、Linux、iOS、Android
2. **抽象层次高**：描述"做什么"，不关心"怎么做"
3. **生态成熟**：支持各种编译器、IDE、测试框架

**适用场景**：

- 跨平台项目
- 复杂的项目结构
- 需要支持多种编译器或 IDE

**不适用场景**：

- 超简单的单文件项目（直接编译更快）
- 只在单平台运行且不需要配置的项目

---

### 为什么组合使用 CMake + Makefile？

**CMake 的痛点**

CMake 的命令又长又难记：

```bash
# 配置 iOS 构建
cmake -B build_ios -S . \
    -DCMAKE_SYSTEM_NAME=iOS \
    -DCMAKE_OSX_ARCHITECTURES=arm64 \
    -DCMAKE_OSX_SYSROOT=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk \
    -DCMAKE_BUILD_TYPE=Release
```

每次要查文档，效率很低。

**Makefile 的优雅**

Makefile 把复杂的 CMake 命令包装成语义化的目标：

```makefile
.PHONY: ios-configure
ios-configure:
    @mkdir -p build_ios
    @cd build_ios && cmake \
        -DCMAKE_SYSTEM_NAME=iOS \
        -DCMAKE_OSX_ARCHITECTURES=$$(uname -m) \
        -DCMAKE_OSX_SYSROOT=$$(xcrun --sdk iphonesimulator --show-sdk-path) \
        ..
```

用户只需要：

```bash
make ios-configure
```

**分层设计**

```text
用户接口层 (Makefile)
    ↓ make ios-build
命令抽象层 (Makefile targets)
    ↓ cmake --build
构建生成层 (CMake)
    ↓ Xcode project / Makefile
平台构建层 (Xcode / make)
    ↓ 编译链接
最终产物 (libmini_react_native.a)
```

每一层都有明确的职责：

- **Makefile**：用户友好的接口
- **CMake**：跨平台的抽象
- **Xcode/make**：实际的构建执行

**什么时候需要这种组合？**

- 项目需要跨平台支持（必须用 CMake）
- 构建命令复杂（需要 Makefile 简化）
- 需要多个构建目标（如 build/test/clean）
- 团队协作需要统一接口

---

## 问题：从单平台到双平台，构建系统要改什么？

当你决定支持第二个平台时，第一个念头可能是：**我要复制一份 Makefile 吗？** 或者 **我要重新配置整个 CMake 吗？**

别慌。来看看 mini-rn 项目从 macOS 单平台到 macOS+iOS 双平台的改动，你会发现——核心改动其实只有 **3 个方向**。

---

## 改动一：Makefile 新增 iOS 构建目标

### 新增的 4 个 iOS 目标

```makefile
# iOS 构建配置（模拟器）
.PHONY: ios-configure
ios-configure:
    @mkdir -p $(BUILD_DIR)_ios
    @cd $(BUILD_DIR)_ios && DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer cmake \
        -DCMAKE_SYSTEM_NAME=iOS \
        -DCMAKE_OSX_ARCHITECTURES=$$(uname -m) \
        -DCMAKE_OSX_SYSROOT=$$(DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer xcrun --sdk iphonesimulator --show-sdk-path) \
        -DCMAKE_BUILD_TYPE=$(CMAKE_BUILD_TYPE) \
        -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
        ..

# 构建 iOS 版本（模拟器）
.PHONY: ios-build
ios-build: js-build ios-configure
    @cd $(BUILD_DIR)_ios && make -j$(CORES)

# iOS 测试目标
.PHONY: ios-test
ios-test: ios-build
    @./test_ios.sh all

# iOS DeviceInfo 测试
.PHONY: ios-test-deviceinfo
ios-test-deviceinfo: ios-build
    @./test_ios.sh deviceinfo
```

### 关键设计决策

**1. 独立构建目录**

macOS 用 `build/`，iOS 用 `build_ios/`，互不干扰：

```makefile
@mkdir -p $(BUILD_DIR)_ios   # iOS 构建目录
```

**2. 仅支持 iOS 模拟器**

为什么不支持真机？因为：

- 真机需要开发者证书和配置文件
- 模拟器足够验证 Bridge 通信机制
- 降低环境配置复杂度

```makefile
-DCMAKE_OSX_SYSROOT=$$(xcrun --sdk iphonesimulator --show-sdk-path)
```

**3. 语义化命令**

`make ios-build` 比写一长串 CMake 命令简洁太多。这就是 Makefile 作为用户接口的价值。

---

## 改动二：CMake 平台条件编译

### 原来的代码（仅 macOS）

```cmake
# 原始版本 - 仅支持 macOS
if(APPLE)
    set(PLATFORM_SOURCES
        src/macos/modules/deviceinfo/DeviceInfoModule.mm
    )
    find_library(IOKIT_FRAMEWORK IOKit)
endif()

target_link_libraries(mini_react_native
    ${JAVASCRIPTCORE_FRAMEWORK}
    ${IOKIT_FRAMEWORK}
)
```

### 演进后的代码（macOS + iOS）

```cmake
# 演进版本 - 支持 macOS + iOS
if(APPLE)
    # 根据具体平台选择源文件
    if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
        set(PLATFORM_SOURCES
            src/ios/modules/deviceinfo/DeviceInfoModule.mm
        )
    else()
        # macOS
        set(PLATFORM_SOURCES
            src/macos/modules/deviceinfo/DeviceInfoModule.mm
        )
    endif()

    # 平台特定框架
    if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
        find_library(UIKIT_FRAMEWORK UIKit)
        set(PLATFORM_FRAMEWORKS ${UIKIT_FRAMEWORK})
    else()
        find_library(IOKIT_FRAMEWORK IOKit)
        set(PLATFORM_FRAMEWORKS ${IOKIT_FRAMEWORK})
    endif()

    # 统一链接
    target_link_libraries(mini_react_native
        ${JAVASCRIPTCORE_FRAMEWORK}
        ${FOUNDATION_FRAMEWORK}
        ${PLATFORM_FRAMEWORKS}
    )
endif()
```

### 三个关键变化

**变化 1：源文件分离**

```text
src/
├── macos/modules/deviceinfo/DeviceInfoModule.mm
└── ios/modules/deviceinfo/DeviceInfoModule.mm
```

两个文件虽然文件名相同，但实现不同：

- **macOS 版本**：用 IOKit 获取硬件信息
- **iOS 版本**：用 UIDevice 获取设备信息

**变化 2：框架动态链接**

| 平台      | 共享框架                      | 平台特定框架 |
|-----------|-------------------------------|--------------|
| macOS     | JavaScriptCore, Foundation    | IOKit        |
| iOS       | JavaScriptCore, Foundation    | UIKit        |

**变化 3：部署目标设置**

```cmake
if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
    set(CMAKE_OSX_DEPLOYMENT_TARGET "12.0")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.15")
endif()
```

---

## 改动三：iOS 特定的资源复制

### 问题：iOS 应用如何读取 JavaScript bundle？

macOS 可以直接从文件系统读取 `dist/bundle.js`，但 iOS 应用有沙盒限制——文件必须在应用包内。

### 解决方案：CMake 自动复制

```cmake
# iOS 特定配置：复制 JavaScript bundle 到应用包
if(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
    add_custom_command(TARGET test_integration POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_SOURCE_DIR}/dist/bundle.js"
        "$<TARGET_FILE_DIR:test_integration>/bundle.js"
        COMMENT "Copying JavaScript bundle to iOS app package"
    )

    # 同时也复制测试脚本
    add_custom_command(TARGET test_integration POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_SOURCE_DIR}/examples/scripts/test_deviceinfo.js"
        "$<TARGET_FILE_DIR:test_integration>/test_deviceinfo.js"
        COMMENT "Copying test script to iOS app package"
    )
endif()
```

### 关键点解读

**1. `POST_BUILD` 钩子**

每次编译后自动执行，不需要手动复制。

**2. `$<TARGET_FILE_DIR:test_integration>`**

这是 CMake 的生成器表达式，自动解析为目标可执行文件所在的目录。不同平台的路径格式不同，CMake 会自动处理。

**3. 对代码透明的资源复制**

JavaScript 代码不需要知道它是从哪里加载的：

- macOS：从文件系统 `dist/bundle.js` 加载
- iOS：从应用包加载

```cpp
// 代码层面无需改动
auto bundlePath = resolveBundlePath();  // 平台适配层处理
```

---

## 额外改进：性能测试可执行文件

diff 还显示了另一个改动：新增了 `test_performance` 可执行文件。

```cmake
# 性能测试可执行文件（轻量级性能检查）
add_executable(test_performance examples/test_performance.cpp)
target_include_directories(test_performance PRIVATE src)
target_link_libraries(test_performance mini_react_native)
```

对应的 Makefile 目标：

```makefile
.PHONY: test-performance
test-performance: build
    @echo "🧪 Running performance test..."
    @./$(BUILD_DIR)/test_performance
    @echo "✅ Performance test complete"
```

**为什么需要性能测试？**

在从单平台到多平台的演进中，性能可能退化：

- 不同平台的编译器优化策略不同
- 框架调用开销可能有差异
- 需要基准数据来对比

---

## 总结：构建系统演进的核心原则

### 1. 单一配置文件，多平台支持

通过 CMake 的条件判断，实现：

- 一套 `CMakeLists.txt` 支持所有平台
- 平台差异通过条件编译处理
- 避免维护多套构建配置

### 2. Makefile 作为统一接口

隐藏 CMake 的复杂性，提供语义化的构建目标：

```makefile
make build           # macOS 构建
make ios-build        # iOS 构建
make test            # 运行测试
make ios-test        # iOS 测试
```

### 3. 资源文件的平台适配

通过 `add_custom_command` 自动处理资源复制，保持代码的跨平台兼容性。

### 4. 测试的逐步完善

从基础测试 → 集成测试 → 性能测试，每个阶段都有对应的构建目标。

---

## 后续扩展的思路

如果未来要支持 Android，只需要：

**1. Makefile 新增目标**

```makefile
.PHONY: android-build
android-build: js-build
    @cd $(BUILD_DIR)_android && cmake --build .
```

**2. CMake 添加 Android 分支**

```cmake
elseif(ANDROID)
    set(PLATFORM_SOURCES
        src/android/modules/deviceinfo/DeviceInfoModule.cpp
    )
    # Android 特定框架（如 JNI）
endif()
```

**3. 资源复制脚本**

```cmake
if(ANDROID)
    add_custom_command(TARGET test_integration POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy ...
    )
endif()
```

框架已经搭好，扩展新平台只是填空题。

---

> 本章通过分析 mini-rn 项目从 macOS 单平台到 macOS+iOS 双平台的构建系统演进，展示了如何通过 CMake + Makefile 的组合，实现优雅的跨平台构建方案。核心思路是：条件编译 + 语义化接口 + 自动化资源处理。
