# SpinalSim 仿真参考手册

> **加载时机**：本手册仅在用户明确要求仿真/验证时加载。架构设计与代码生成阶段无需加载。
> **源码依据**：`core/src/main/scala/spinal/core/sim/`（package.scala、SimBootstraps.scala、ClockDomain.scala）+ `lib/src/main/scala/spinal/lib/sim/`

## 1. 仿真配置与启动

### 1.1 SimConfig — 仿真入口

```scala
import spinal.core.sim._

SimConfig
  .withWave               // 记录波形（默认 VCD）
  .withWaveDepth(2)       // 波形层次深度（0=全部）
  .compile(new MyTop)     // 编译 DUT（Verilator 后端）
  .doSim { dut =>
    dut.clockDomain.forkStimulus(10)   // 10 时间单位周期的时钟
    // 测试主体
  }
```

### 1.2 SpinalSimConfig 完整 API

**后端选择**（默认 Verilator）：

| 方法 | 后端 | 说明 |
|------|------|------|
| `.withVerilator` | Verilator | C++ 编译，最快，`'x'/'z'` 随机化 |
| `.withGhdl()` | GHDL | VHDL 仿真（生成 VHDL） |
| `.withIVerilog` | Icarus Verilog | 开源 VPI 仿真 |
| `.withVCS` / `.withVCS(flags)` | Synopsys VCS | 商业仿真 |
| `.withXSim` | Xilinx XSim | Vivado 仿真（支持 xci/bd） |

**波形控制**：

| 方法 | 说明 |
|------|------|
| `.withWave` | 默认格式（Verilator→VCD，VCS→VPD） |
| `.withWave(depth: Int)` | 波形 + 深度限制 |
| `.withVcdWave` / `.withFstWave` / `.withFsdbWave` / `.withVPDWave` | 指定格式 |
| `.withWaveDepth(depth)` | 波形层次深度（0=全深度） |
| `.waveFilePrefix(prefix)` | 波形文件名前缀 |

**编译与工作区**：

| 方法 | 说明 |
|------|------|
| `.workspacePath(path)` | 工作区路径（默认 `./simWorkspace`，或 `SPINALSIM_WORKSPACE` 环境变量） |
| `.workspaceName(name)` | 工作区名称（默认 toplevel 名） |
| `.withConfig(spinalConfig)` | 传入 SpinalConfig |
| `.addRtl(path)` / `.addIncludeDir(dir)` | 附加 Verilog 源/头文件 |
| `.addSimulatorFlag(flag)` / `.addRunFlag(flag)` | 传给仿真器/运行时标志 |
| `.noOptimisation` / `.fewOptimisation` / `.normalOptimisation` / `.allOptimisation` | Verilator 优化等级（默认 0） |
| `.disableCache` | 禁用编译缓存 |
| `.withCoverage` | Verilator 覆盖率 |
| `.withTimeScale(t)` / `.withTimePrecision(t)` / `.withTimeSpec(scale, precision)` | 时间单位/精度 |
| `.setTestPath(path)` | 测试输出路径（`$WORKSPACE`/`$COMPILED`/`$TEST` 变量） |

### 1.3 doSim 系列

| 方法 | 说明 |
|------|------|
| `.doSim { dut => ... }` | 运行仿真（body 线程结束后自动结束） |
| `.doSim("testName") { dut => ... }` | 指定测试名（用于波形文件命名） |
| `.doSim(seed) { dut => ... }` | 指定随机种子 |
| `.doSim("name", seed) { dut => ... }` | 同时指定 |
| `.doSimUntilVoid { dut => ... }` | 等所有 fork 线程结束后才结束（用于自驱动测试） |

> **seed 规则**：默认随机 seed；可用 `SPINAL_SIM_SEED` 环境变量固定。仿真内 `simRandom` 与该 seed 关联，保证可复现。

## 2. 信号读写

### 2.1 赋值（# =）

```scala
dut.io.valid #= true                    // Bool ← Boolean
dut.io.payload #= 42                    // UInt/Bits ← Int/Long
dut.io.payload #= BigInt("DEADBEEF", 16)// 大位宽 ← BigInt
dut.io.payload #= Array[Byte](...)      // BitVector ← 字节数组
dut.myEnum #= MyEnum.sIDLE              // SpinalEnum ← 枚举元素
dut.myUfix #= 3.14                      // UFix/SFix/AFix ← Double/BigDecimal
```

### 2.2 读取

```scala
dut.io.valid.toBoolean                  // Bool → Boolean
dut.io.payload.toInt                    // BitVector → Int
dut.io.payload.toLong                   // BitVector → Long
dut.io.payload.toBigInt                 // BitVector → BigInt
dut.myEnum.toEnum                       // → SpinalEnumElement
dut.myUfix.toBigDecimal                 // UFix/SFix/AFix → BigDecimal
```

### 2.3 随机化

```scala
dut.io.payload.randomize()              // 随机赋值，返回随机值
dut.io.someBundle.randomize()           // Data.randomize → 递归随机化所有叶子
simRandom.nextInt(100)                  // 专用随机源（与 seed 关联）
```

### 2.4 访问内部信号 — simPublic

```scala
// 硬件侧（生成阶段）：
val internalCounter = Reg(UInt(8 bits))
internalCounter.simPublic()             // 暴露给仿真
val mem = Mem(Bits(32 bits), 1024)
mem.simPublic()                         // 暴露 Mem

// 仿真侧（doSim 内）：
val c = dut.internalCounter.toInt       // 直接读写

// 错误提示：未 simPublic 的信号访问会报 UNACCESSIBLE SIGNAL
```

### 2.5 Mem 仿真读写

```scala
dut.mem.simPublic()                     // 硬件侧先暴露
// 仿真侧（doSim 内）：
dut.mem.setBigInt(address = 5, data = 0x1234)  // 写
val v = dut.mem.getBigInt(address = 5)         // 读
```

## 3. 时钟域 API

### 3.1 时钟激励

```scala
dut.clockDomain.forkStimulus(10)        // 周期 10 时间单位（默认复位 16 周期）
dut.clockDomain.forkStimulus(10 MHz)    // 按频率
dut.clockDomain.forkStimulus()          // 需 ClockDomain 有 FixedFrequency
```

**复位行为**：`forkStimulus` 自动处理复位——ASYNC 复位先拉高 16 周期再释放；SYNC 复位在时钟运行中保持 16 周期；BOOT 复位无信号。

### 3.2 等待原语

| 方法 | 说明 |
|------|------|
| `cd.waitSampling()` / `cd.waitSampling(n)` | 等有效采样沿（复位释放 + 时钟使能 + 有效边沿） |
| `cd.waitSamplingWhere(cond)` | 等满足条件的采样沿 |
| `cd.waitSamplingWhere(timeout)(cond)` | 带超时（返回 true=超时） |
| `cd.waitRisingEdge()` / `cd.waitRisingEdge(n)` | 等上升沿（不受复位影响） |
| `cd.waitFallingEdge(n)` / `cd.waitEdge(n)` | 等下降沿/任意沿 |
| `cd.waitActiveEdge(n)` | 等 ClockDomain 配置的有效沿 |
| `cd.waitRisingEdgeWhere(cond)` | 等满足条件的上升沿 |

### 3.3 回调注册（无线程开销）

| 方法 | 说明 |
|------|------|
| `cd.onSamplings { ... }` | 每个有效采样沿执行（多回调共享一个敏感器） |
| `cd.onNextSampling { ... }` | 仅下一个采样沿执行一次 |
| `cd.onSamplingWhile { ... }` | 反复执行直到返回 false |
| `cd.onRisingEdges` / `cd.onFallingEdges` / `cd.onEdges` / `cd.onActiveEdges` | 对应边沿回调 |

### 3.4 复位/使能控制

```scala
dut.clockDomain.assertReset()           // 拉高复位
dut.clockDomain.deassertReset()         // 释放复位
dut.clockDomain.isResetAsserted         // 查询复位状态
dut.clockDomain.isSamplingEnable        // 复位释放且时钟使能有效
dut.clockDomain.forkSimSpeedPrinter()   // 每秒打印仿真速度（k cycles/s）
```

## 4. 线程与调度

```scala
fork {                                  // 创建新仿真线程
  while(true) { dut.io.x #= simRandom.nextBoolean(); cd.waitSampling() }
}

forkJoin(task1 _, task2 _)              // 并行执行并等待全部完成

sleep(100)                              // 线程睡眠 100 时间单位
sleep(1 ms)                             // 支持时间字面量

waitUntil(dut.io.done.toBoolean)        // 等条件为真（忙等敏感）

forkSensitive { ... }                   // 值变化即执行（无线程）
forkSensitiveWhile { cond }             // 值变化执行直到返回 false
forkSensitive(dut.io.x) { ... }         // 指定信号变化触发
forkSensitive2(dut.io.a, dut.io.b) { ... }  // 多信号触发

delayed(100) { ... }                    // 100 时间单位后执行一次
periodically(10) { ... }                // 每 10 时间单位执行

simTime()                               // 当前仿真时间
simDeltaCycle()                         // 当前 delta cycle 数
```

**线程 vs 敏感器**：`fork`+`wait` 是线程 API（直观但慢）；`forkSensitive`/`onSamplings` 是无线程 API（快）。`StreamDriver`/`StreamMonitor` 内部用无线程实现。

## 5. 仿真流程控制

```scala
simSuccess()                            // 立即成功结束
simFailure("msg")                       // 立即失败结束
SimTimeout(10000)                       // 10000 时间单位后超时失败（防挂死）
SimTimeout(1 ms)                        // 时间字面量

val mutex = SimMutex()                  // 仿真互斥锁
mutex.lock() / mutex.unlock() / mutex.await()
val mutex = SimMutex(randomized = true) // 随机唤醒顺序（测试调度鲁棒性）
```

## 6. Stream 仿真库（spinal.lib.sim）

### 6.1 StreamDriver — 驱动 valid/payload

```scala
import spinal.lib.sim._

// 方式 1：回调驱动（返回 true 表示 payload 有效）
StreamDriver(dut.io.input, dut.clockDomain) { payload =>
  if (queue.nonEmpty) { payload #= queue.dequeue(); true }
  else false                            // false → valid 拉低等待
}

// 方式 2：队列驱动
val (driver, cmdQueue) = StreamDriver.queue(dut.io.input, dut.clockDomain)
cmdQueue.enqueue(p => p.data #= 0xAB)   // 依次入队

// 方式 3：从集合驱动
StreamDriver.fromIterable(dut.io.input, dut.clockDomain, Seq(1, 2, 3))
```

**驱动节奏控制**：

```scala
val driver = StreamDriver(dut.io.input, dut.clockDomain) { p => ... }
driver.setFactor(0.5f)                  // 每周期 50% 概率发送（1.0=每拍发）
driver.setFactorPeriodically(100)        // 每 100 周期随机化概率
driver.transactionDelay = () => 5        // 自定义间隔周期数
```

### 6.2 StreamMonitor — 监视 fire 事件

```scala
StreamMonitor(dut.io.output, dut.clockDomain) { payload =>
  println(s"fire: ${payload.toInt}")
  score.pushRef(payload.toInt)          // 典型用法：接入记分板
}
```

### 6.3 StreamReadyRandomizer — 随机化 ready

```scala
StreamReadyRandomizer(dut.io.output, dut.clockDomain)
  .setFactor(0.7f)                      // 70% 概率 ready
// 或带条件：
StreamReadyRandomizer(dut.io.output, cd) { () => someCondition }
```

### 6.4 StreamDriverOoo — 乱序突发驱动

```scala
val ooo = StreamDriverOoo(dut.io.cmd, dut.clockDomain)
ooo.burst { push =>
  push { p => p.addr #= 0x1000 }        // 突发内保序
  push { p => p.addr #= 0x1004 }
}
ooo.single { p => p.addr #= 0x2000 }    // 单事务
// 多个 burst/single 之间乱序调度
```

### 6.5 SimStreamAssert — 协议断言

```scala
new SimStreamAssert(dut.io.stream, dut.clockDomain)
// 断言：valid 保持期间 payload 不变；ready 后 valid 才可变
```

## 7. Flow 仿真库

```scala
FlowDriver(dut.io.input, dut.clockDomain) { payload => ... }
FlowDriver.queue(flow, cd)               // 队列驱动
FlowDriver.fromIterable(flow, cd, Seq(...))
FlowMonitor(dut.io.output, dut.clockDomain) { payload => ... }
```

## 8. ScoreboardInOrder — 有序记分板

```scala
val score = ScoreboardInOrder[Int]()

// 参考模型侧（输入监视）
StreamMonitor(dut.io.input, dut.clockDomain) { p =>
  score.pushRef(myModel(p.toInt))        // 参考值入队
}

// DUT 侧（输出监视）
StreamMonitor(dut.io.output, dut.clockDomain) { p =>
  score.pushDut(p.toInt)                 // DUT 值入队，两侧对头即比对
}

// 结束检查（配合 Phase 或手动调用）
score.checkEmptiness()                   // 队列非空 → 失败并打印差异
```

**比对语义**：`pushRef`+`pushDut` 各自入队，两侧都有数据时立即比较队头；不匹配打印 REF/DUT 并 `simFailure`。

## 9. Phase — 阶段化管理

```scala
import spinal.lib.sim.Phase

Phase.boot()                             // 在 doSim 体内首先调用
Phase.setup { /* 初始化：Fork 时钟、驱动 */ }
Phase.stimulus { /* 激励主体 */ }
Phase.flush { /* 排空 FIFO 等残余 */ }
Phase.check { /* 终检：score.checkEmptiness() */ }
// check 阶段全部线程结束后自动进入 end 阶段并 simSuccess（object Phase.end 为 private，不直接调用）

// Phase 内的线程自动被追踪，全部完成后进入下一阶段
Phase.stimulus.retainer(1).release()     // 手动控制阶段推进
```

**阶段顺序**：`setup → stimulus → flush → check → end`。未使用 Phase 时需手动调用 `checkEmptiness()`。

## 10. SparseMemory 与区域分配

```scala
// SparseMemory 位于 spinal.lib.bus.amba4.axi.sim（非 spinal.lib.sim）
import spinal.lib.bus.amba4.axi.sim.SparseMemory

val mem = SparseMemory()
mem.write(0x1000, 0x12.toByte)                   // 单字节写（按 1MB 页分配）
mem.read(0x1000)                                 // → Byte
mem.writeBigInt(0x1000, BigInt(0x12345678), width = 4)  // 多字节写（小端）
mem.readBigInt(0x1000, length = 4)               // → BigInt
mem.writeArray(0x2000, byteArray)                // 字节数组写
mem.readArray(0x2000, 4)                         // → Array[Byte]
mem.loadBinary(0x0, "boot.bin")                  // 加载二进制文件

// MemoryRegionAllocator 位于 spinal.lib.sim
import spinal.lib.sim.MemoryRegionAllocator
val alloc = MemoryRegionAllocator(base = 0x10000000L, size = 0x10000)
val region = alloc.allocate(size = 256)          // 随机分配 SizeMapping
alloc.free(region)
alloc.isAllocated(addr)
```

## 11. SimData — 结构化快照

```scala
val snapshot = SimData.copy(dut.io.someBundle)
snapshot("fieldA")                       // 读字段值（BigInt）
snapshot("fieldA") = 42                  // 写字段值
snapshot.check(dut.io.someBundle)        // 与当前硬件值比对 → Boolean
snapshot.write(dut.io.someBundle)        // 将快照写回硬件
```

## 12. 完整示例

### 12.1 StreamFifo 仿真（driver + monitor + scoreboard）

```scala
import spinal.core._
import spinal.core.sim._
import spinal.lib._
import spinal.lib.sim._

object StreamFifoSim extends App {
  SimConfig.withWave.compile(new StreamFifo(Bits(8 bits), depth = 16)).doSim { dut =>
    dut.clockDomain.forkStimulus(10)
    SimTimeout(100000)                   // 防挂死

    val score = ScoreboardInOrder[Int]()

    // 输入驱动：随机数据
    StreamDriver(dut.io.push, dut.clockDomain) { p =>
      p.randomize(); true                // 每拍都发
    }

    // 随机反压
    StreamReadyRandomizer(dut.io.pop, dut.clockDomain).setFactor(0.5f)

    // 输入监视 → 参考值
    StreamMonitor(dut.io.push, dut.clockDomain) { p =>
      score.pushRef(p.toInt)
    }
    // 输出监视 → DUT 值
    StreamMonitor(dut.io.pop, dut.clockDomain) { p =>
      score.pushDut(p.toInt)
    }

    // 运行足够周期
    dut.clockDomain.waitSampling(200)
    score.checkEmptiness()               // 残余事务 → 失败
  }
}
```

### 12.2 复位后检查 + 内部信号访问

```scala
SimConfig.compile(new MyDut).doSim { dut =>
  dut.clockDomain.forkStimulus(10)

  // 主动复位测试
  dut.clockDomain.assertReset()
  dut.clockDomain.waitSampling(5)
  dut.clockDomain.deassertReset()
  dut.clockDomain.waitSamplingWhere(dut.io.busy.toBoolean == false)

  // 内部计数器检查（需硬件侧 simPublic()）
  waitUntil(dut.internalCounter.toInt == 100)
  simSuccess()
}
```

## 13. 常见陷阱

| 陷阱 | 对策 |
|------|------|
| 访问内部信号报 UNACCESSIBLE SIGNAL | 硬件侧调用 `.simPublic()`，并确保信号有名字 |
| 赋值在 `forkStimulus` 之前 | 先 `forkStimulus`（内部会给时钟/复位初值）再驱动信号 |
| `doSim` body 结束但 fork 线程未完 | 用 `doSimUntilVoid`（等全部线程） |
| 波形无信号 | `.withWave` + `withWaveDepth(0)` 全深度 |
| 复位期间采样无效 | `waitSampling` 不含复位期；用 `waitRisingEdge` 跳过复位 |
| `simRandom` 与全局 Random 混用 | 仿真内只用 `simRandom`（seed 关联可复现） |
| 仿真挂死无提示 | 在开头加 `SimTimeout(周期数)` |
