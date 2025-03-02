## **1 背景**

当前，Kmesh 需要一个轻量级的单元测试框架来测试 eBPF 程序。该框架应能够独立运行单个 eBPF 程序的测试，而无需加载整个 Kmesh 系统，从而提高测试效率和覆盖率。

## **2 测试框架设计**

### 2.1 核心组件

测试框架由三个核心组件组成：

1. 测试管理 (common.h)
    - 测试套件管理
    - 测试用例执行 
    - 结果收集和报告

2. XDP 测试运行时 (xdp_test.c)
    - eBPF 程序加载
    - 数据包构造和注入
    - 测试结果验证

3. 构建系统 (Makefile)
    - 编译 eBPF 程序
    - 生成测试框架
    - 链接所需依赖

### 2.2 测试结构

#### 2.2.1 测试套件

测试套件使用以下结构来管理测试状态：

```c
typedef struct {
     const char *suite_name;
     test_context_t subtests[MAX_SUBTESTS];
     int subtest_count;
     int passed_count;
     int failed_count;
     int skipped_count;
} test_suite_t;
```

#### 2.2.2 测试用例

每个测试用例包含以下信息：

```c
typedef struct {
     const char *name;
     test_status_t status;
     int result;
     const char *message;
     double duration;
} test_context_t;
```

### 2.3 关键函数

#### 2.3.1 测试初始化和清理

```c
// Initialize test suite
test_init("test_suite_name");
```

test_init 函数初始化测试套件，设置测试环境和计数器：

```c
static inline void test_init(const char *test_name) {
     printf("\n=== Starting test suite: %s ===\n", test_name);
     current_suite.suite_name = test_name;
     current_suite.subtest_count = 0;
     current_suite.passed_count = 0;
     current_suite.failed_count = 0;
     current_suite.skipped_count = 0;
}
```

主要功能：
- 初始化测试套件名称
- 重置所有计数器（总计、通过、失败、跳过）
- 打印测试套件启动信息

**test_finish();**

```c
static inline void test_finish(void) {
     printf("\n=== Test suite summary: %s ===\n", current_suite.suite_name);
     printf("Total tests: %d\n", current_suite.subtest_count);
     printf("  Passed:  %d\n", current_suite.passed_count);
     printf("  Failed:  %d\n", current_suite.failed_count);
     printf("  Skipped: %d\n", current_suite.skipped_count);
     
     // Print detailed results
     if (current_suite.subtest_count > 0) {
          printf("\nDetailed results:\n");
          for (int i = 0; i < current_suite.subtest_count; i++) {
                test_context_t *test = &current_suite.subtests[i];
                // ... Print results for each test case ...
          }
     }
}
```

主要功能：
- 打印测试套件摘要
- 显示测试统计信息
- 输出每个测试用例的详细结果，包括：
  - 测试名称
  - 执行状态（通过/失败/跳过）
  - 执行时间
  - 错误消息（如果有）

#### 2.3.2 测试用例定义

使用 TEST 宏定义测试用例：

TEST 宏为定义和执行单个测试用例提供框架：

```c
#define TEST(test_name, fn) \
     do { \
          // 1. Initialize test context
          test_context_t *_test_ctx = &current_suite.subtests[current_suite.subtest_count++]; \
          _test_ctx->name = test_name; \
          _test_ctx->status = TEST_STATUS_RUNNING; \
          _test_ctx->result = TEST_PASS; \
          
          // 2. Record start time
          struct timespec _start_time, _end_time; \
          clock_gettime(CLOCK_MONOTONIC, &_start_time); \
          
          // 3. Execute test
          test_log("\n--- Starting test: %s ---", test_name); \
          fn(); \
          
          // 4. Calculate execution time
          clock_gettime(CLOCK_MONOTONIC, &_end_time); \
          _test_ctx->duration = (_end_time.tv_sec - _start_time.tv_sec) + \
                                      (_end_time.tv_nsec - _start_time.tv_nsec) / 1e9; \
          
          // 5. Update test status
          _test_ctx->status = TEST_STATUS_COMPLETED; \
          switch (_test_ctx->result) { \
                case TEST_PASS: current_suite.passed_count++; break; \
                case TEST_FAIL: current_suite.failed_count++; break; \
                case TEST_SKIP: current_suite.skipped_count++; break; \
          } \
     } while(0)
```

主要功能：
- 测试上下文管理：
  - 创建新的测试上下文
  - 设置初始状态和结果
- 时间跟踪：
  - 记录开始和结束时间
  - 计算测试执行时间
- 状态管理：
  - 更新测试状态
  - 维护测试计数器
- 日志记录：
  - 记录测试开始和结束
  - 输出测试结果

#### 2.3.3 测试跳过机制 (SKIP_SUB_TEST)

SKIP_SUB_TEST 宏允许在运行时动态跳过测试：

```c
#define SKIP_SUB_TEST(msg) \
     do { \
          test_log("Skipping test: %s", msg); \
          current_test_ctx->result = TEST_SKIP; \
          current_test_ctx->message = msg; \
          break; \
     } while(0)
```

主要功能：
- 将测试标记为已跳过
- 记录跳过原因
- 提前终止测试

#### 2.3.4 断言机制

```c
test_assert(condition, "error message");
```

test_assert 宏提供测试验证功能：

```c
#define test_assert(cond, msg) \
     do { \
          if (!(cond)) { \
                test_log("Assert failed: %s", msg); \
                test_log("At %s:%d", __FILE__, __LINE__); \
                if (current_test_ctx) { \
                     current_test_ctx->result = TEST_FAIL; \
                     current_test_ctx->message = msg; \
                } \
                return; \
          } \
     } while(0)
```

主要功能：
- 条件验证：
  - 检查指定条件是否为真
  - 在失败时记录详细信息
- 错误处理：
  - 将测试状态更新为失败
  - 记录失败消息和位置
  - 终止测试执行

## **3 XDP 测试实现**

### 3.1 测试环境设置

```c
int main() {
     test_init("xdp_test");
     
     TEST("BPF Program Load", bpf_load);
     TEST("Packet Parsing", test_packet_parsing);
     TEST("IP Version Check", test_ip_version_check);
     TEST("Tuple Extraction", test_tuple_extraction);
     TEST("Connection Shutdown", test_connection_shutdown);
     TEST("BPF Program Cleanup", bpf_offload);

     test_finish();
     return current_suite.failed_count > 0 ? 1 : 0;
}
```

### 3.2 测试用例示例

#### 3.2.1 基本数据包解析测试

```c
void test_packet_parsing() {
     unsigned char packet[PACKET_SIZE] = {0};
     struct ethhdr *eth = (struct ethhdr *)packet;
     struct iphdr *ip = (struct iphdr *)(packet + sizeof(struct ethhdr));
     
     // Set test data
     eth->h_proto = htons(ETH_P_IP);
     ip->version = 4;
     // ... More configuration ...
     
     // Run test
     int err = run_xdp_test(packet, PACKET_SIZE);
     test_assert(err == 0, "run_xdp_test failed");
}
```

#### 3.2.2 连接关闭测试

```c
void test_connection_shutdown() {
     // Prepare test packet
     unsigned char packet[PACKET_SIZE] = {0};
     // ... Configure headers ...
     
     // Configure test conditions
     struct bpf_sock_tuple tuple = { /* ... */ };
     __u32 value = AUTH_FORBID;
     
     // Verify results
     test_assert(modified_tcp->rst == 1, "RST flag not set");
     test_assert(modified_tcp->syn == 0, "SYN flag not cleared");
}
```

## **4 使用方法**

### 4.1 编写测试

1. 创建测试文件 (例如：xdp_test.c)
2. 包含所需头文件：
```c
#include "common.h"
#include "xdp_test.skel.h"
```

3. 实现测试用例：
```c
int main() {
     test_init("xdp_test");
     
     TEST("BPF Program Load", bpf_load);
     TEST("Packet Parsing", test_packet_parsing);
     // ... More tests ...
     
     test_finish();
     return current_suite.failed_count > 0 ? 1 : 0;
}
```

### 4.2 运行测试

1. 编译测试程序：
```bash
make xdp_test
```

2. 执行测试：
```bash
./xdp_test
```

3. 结果：

![xdp_test_result1](./pics/xdp_test_result1.png)

![xdp_test_result2](./pics/xdp_test_result2.png)
