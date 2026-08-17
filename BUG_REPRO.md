# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

频轨授权在并发申请下大面积丢账：申请成功了，但授权台账里查不到那条记录。

```
$ ./satctl spectrum allocate --workers 64 --per-worker 5 --band ka --mhz 10 --rounds 20
{
  "band": "ka",
  "total_mhz": 1000,
  "rounds": 20,
  "inconsistent_rounds": 20,
  "lost_grants": 677,
  "ok": false,
  ...
}
错误: model: 频段超额分配: 20/20 轮出现不一致, 累计丢失授权记录 677 条
$ echo $?
6
```

这条命令跑 20 轮，每轮都用一个全新的分配器、64 个协程各申请 5 次、每次 10MHz。按 README，成功申请的次数必须等于授权记录条数，已授权带宽必须等于全部授权记录之和且不超过总带宽。实际 20 轮全部不一致，平均每轮丢掉三十多条授权。

逐轮明细里能看到 `granted_calls` 和 `grant_records` 对不上，`used_mhz` 也跟 `expected_used_mhz` 不符，有的轮次 `oversubscribed` 还是 true —— 也就是已授权带宽超过了该频段总带宽 1000MHz。

`go test -race ./...` 会直接报数据竞争。

对照现象：

- `--workers 1`（完全串行）时每轮都准确，`inconsistent_rounds` 是 0。
- 协程数越多丢得越多；同一条命令重复跑，丢的条数每次都不一样。
- 带宽耗尽时的拒绝逻辑本身是对的：串行跑到上限后继续申请会被正常拒绝并计入 `rejects`。

帮我修好，让并发申请下的授权台账严格自洽。已有测试跑一遍不要有回归，`-race` 也要干净。

## 含 Bug 版本

- 仓库：VanceMichael/go-annotation-36
- 仓库地址：https://github.com/VanceMichael/go-annotation-36.git
- parent SHA：8836fbb83ee11cd8f7e3ba9cca2714a89d20c354

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-annotation-36.git bug-repro
cd bug-repro
git checkout --detach 8836fbb83ee11cd8f7e3ba9cca2714a89d20c354
go test -race ./internal/spectrum/ ./internal/cli/ -run "TestConcurrentAllocateKeepsGrantsConsistent|TestConcurrentAllocateNeverOversubscribes|TestAllocateVerifyAfterConcurrentLoad|TestCLISpectrumAllocateConsistent" -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test -race ./internal/spectrum/ ./internal/cli/ -run "TestConcurrentAllocateKeepsGrantsConsistent|TestConcurrentAllocateNeverOversubscribes|TestAllocateVerifyAfterConcurrentLoad|TestCLISpectrumAllocateConsistent" -count=1
==================
WARNING: DATA RACE
Read at 0x00c0000b0108 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9b4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b0108 by goroutine 38:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 71 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 38 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c0000b24d8 by goroutine 38:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous read at 0x00c0000b24d8 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:62 +0x504
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 38 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 71 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c0000b00f0 by goroutine 10:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xb9b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b00f0 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 10 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 8 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c0000b24d8 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb35
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b24d8 by goroutine 38:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 71 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 38 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c0000b24d8 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b24d8 by goroutine 28:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 71 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 28 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c0000b0108 by goroutine 52:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b0108 by goroutine 28:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 52 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 28 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c0000b0108 by goroutine 52:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:70 +0xa04
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b0108 by goroutine 64:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 52 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 64 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c000180bf8 by goroutine 52:
  runtime.growslice()
      /usr/local/go/src/runtime/slice.go:155 +0x0
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xbd7
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c000180bf8 by goroutine 26:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 52 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 26 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c0000b00f0 by goroutine 28:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0000b00f0 by goroutine 52:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 28 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 52 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c00028c820 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c00028c820 by goroutine 28:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 71 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 28 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x1fa
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
--- FAIL: TestConcurrentAllocateKeepsGrantsConsistent (0.02s)
    spectrum_test.go:89: 第 1/20 轮: 成功申请 320 次, 授权记录只有 317 条（丢失 3 条）
    testing.go:1398: race detected during execution of test
==================
WARNING: DATA RACE
Read at 0x00c0005b40b0 by goroutine 86:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x54a
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Previous write at 0x00c0005b40b0 by goroutine 85:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x56b
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0x138
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x41

Goroutine 86 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x1bd
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 85 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xda
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x1bd
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
--- FAIL: TestConcurrentAllocateNeverOversubscribes (0.00s)
    spectrum_test.go:110: 并发分配后应自洽: model: 频段超额分配: Ku频段 已授权 500MHz, 授权记录合计 90MHz
    testing.go:1398: race detected during execution of test
--- FAIL: TestAllocateVerifyAfterConcurrentLoad (0.00s)
    spectrum_test.go:118: 并发分配后核对失败: model: 频段超额分配: Ka频段 已授权 512MHz, 授权记录合计 160MHz
FAIL
FAIL	satnet/internal/spectrum	0.186s
==================
WARNING: DATA RACE
Read at 0x00c000136168 by goroutine 26:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9b4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000136168 by goroutine 19:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 26 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 19 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c000142798 by goroutine 19:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous read at 0x00c000142798 by goroutine 32:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:62 +0x504
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 19 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 32 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c000136168 by goroutine 32:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000136168 by goroutine 28:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 32 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 28 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c000136150 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous read at 0x00c000136150 by goroutine 19:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xb9b
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 19 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c000136168 by goroutine 32:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:70 +0xa04
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000136168 by goroutine 30:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x9d5
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 32 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 30 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c000316000 by goroutine 30:
  runtime.growslice()
      /usr/local/go/src/runtime/slice.go:155 +0x0
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xbd7
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000316000 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 30 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Read at 0x00c000142798 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb35
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000142798 by goroutine 12:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 16 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 12 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c000136150 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000136150 by goroutine 27:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xca4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 27 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c000142798 by goroutine 27:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c000142798 by goroutine 30:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0xb5b
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 27 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 30 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
==================
WARNING: DATA RACE
Write at 0x00c0000ec6e0 by goroutine 105:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Previous write at 0x00c0000ec6e0 by goroutine 118:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0xc24
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x216
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x41

Goroutine 105 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44

Goroutine 118 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1b84
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x45d
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xf8
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x1e7
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x244
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x21e
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x44
==================
--- FAIL: TestCLISpectrumAllocateConsistent (0.17s)
    app_test.go:94: 并发分配退出码应为 0, 实际 6
        错误: model: 频段超额分配: 5/10 轮出现不一致, 累计丢失授权记录 208 条
    testing.go:1398: race detected during execution of test
FAIL
FAIL	satnet/internal/cli	0.243s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test -race ./internal/spectrum/ ./internal/cli/ -run "TestConcurrentAllocateKeepsGrantsConsistent|TestConcurrentAllocateNeverOversubscribes|TestAllocateVerifyAfterConcurrentLoad|TestCLISpectrumAllocateConsistent" -count=1
==================
WARNING: DATA RACE
Read at 0x00c00001a4f8 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:62 +0x37c
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00001a4f8 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 8 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 71 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c000072210 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x830
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c000072210 by goroutine 71:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 8 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 71 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c000072228 by goroutine 41:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x704
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c000072228 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 41 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 8 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c00007ebf8 by goroutine 17:
  runtime.growslice()
      /usr/local/go/src/runtime/slice.go:155 +0x0
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x868
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00007ebf8 by goroutine 38:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 17 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 38 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c000072228 by goroutine 40:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c000072228 by goroutine 45:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 40 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 45 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c000072228 by goroutine 40:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:70 +0x734
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c000072228 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 40 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 16 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c000072210 by goroutine 13:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c000072210 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 13 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 16 (finished) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c00001a4f8 by goroutine 22:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x7f0
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00001a4f8 by goroutine 49:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 22 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 49 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c00001a4f8 by goroutine 37:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00001a4f8 by goroutine 49:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 37 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 49 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateKeepsGrantsConsistent()
      /app/internal/spectrum/spectrum_test.go:87 +0x194
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
--- FAIL: TestConcurrentAllocateKeepsGrantsConsistent (0.01s)
    spectrum_test.go:89: 第 1/20 轮: 成功申请 320 次, 授权记录只有 293 条（丢失 27 条）
    testing.go:1398: race detected during execution of test
==================
WARNING: DATA RACE
Read at 0x00c00031a410 by goroutine 101:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x3ac
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00031a410 by goroutine 95:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x3c0
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 101 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x170
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 95 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x170
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c00031a410 by goroutine 123:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x3c0
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00031a410 by goroutine 130:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:63 +0x3c0
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 123 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x170
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 130 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestConcurrentAllocateNeverOversubscribes()
      /app/internal/spectrum/spectrum_test.go:104 +0x170
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
--- FAIL: TestConcurrentAllocateNeverOversubscribes (0.00s)
    spectrum_test.go:107: 已授权 520MHz 超过总带宽 500MHz
    testing.go:1398: race detected during execution of test
==================
WARNING: DATA RACE
Write at 0x00c00037efc8 by goroutine 154:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Previous write at 0x00c00037efc8 by goroutine 157:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/spectrum_test.concurrentAllocate.func1()
      /app/internal/spectrum/spectrum_test.go:70 +0xf0
  satnet/internal/spectrum_test.concurrentAllocate.gowrap1()
      /app/internal/spectrum/spectrum_test.go:76 +0x44

Goroutine 154 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestAllocateVerifyAfterConcurrentLoad()
      /app/internal/spectrum/spectrum_test.go:116 +0x178
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 157 (running) created at:
  satnet/internal/spectrum_test.concurrentAllocate()
      /app/internal/spectrum/spectrum_test.go:67 +0xb8
  satnet/internal/spectrum_test.TestAllocateVerifyAfterConcurrentLoad()
      /app/internal/spectrum/spectrum_test.go:116 +0x178
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
--- FAIL: TestAllocateVerifyAfterConcurrentLoad (0.00s)
    spectrum_test.go:118: 并发分配后核对失败: model: 频段超额分配: Ka频段 已授权 458MHz, 授权记录合计 456MHz
    testing.go:1398: race detected during execution of test
FAIL
FAIL	satnet/internal/spectrum	0.022s
==================
WARNING: DATA RACE
Read at 0x00c0000f2798 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:62 +0x37c
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000f2798 by goroutine 39:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 39 (finished) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c0000aa210 by goroutine 8:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x830
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000aa210 by goroutine 39:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 39 (finished) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c0000fe268 by goroutine 8:
  runtime.growslice()
      /usr/local/go/src/runtime/slice.go:155 +0x0
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x868
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000fe268 by goroutine 39:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 8 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 39 (finished) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c0000aa228 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x704
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000aa228 by goroutine 25:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 24 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 25 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c0000aa228 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000aa228 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 24 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 16 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Read at 0x00c0000f2798 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x7f0
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000f2798 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 24 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 16 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c0000f2798 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000f2798 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:75 +0x808
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 24 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 16 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c0000aa228 by goroutine 25:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:68 +0x718
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous read at 0x00c0000aa228 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:70 +0x734
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 25 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 24 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c0000aa210 by goroutine 16:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c0000aa210 by goroutine 24:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x8e4
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 16 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 24 (finished) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
==================
WARNING: DATA RACE
Write at 0x00c000191690 by goroutine 118:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Previous write at 0x00c000191690 by goroutine 119:
  satnet/internal/spectrum.(*Allocator).Allocate()
      /app/internal/spectrum/spectrum.go:76 +0x898
  satnet/internal/cli.(*app).runSpectrum.func1()
      /app/internal/cli/app.go:513 +0x188
  satnet/internal/cli.(*app).runSpectrum.gowrap1()
      /app/internal/cli/app.go:519 +0x44

Goroutine 118 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40

Goroutine 119 (running) created at:
  satnet/internal/cli.(*app).runSpectrum()
      /app/internal/cli/app.go:510 +0x1574
  satnet/internal/cli.(*app).route()
      /app/internal/cli/app.go:135 +0x480
  satnet/internal/cli.Run()
      /app/internal/cli/app.go:93 +0xa4
  satnet/internal/cli_test.run()
      /app/internal/cli/app_test.go:15 +0x198
  satnet/internal/cli_test.TestCLISpectrumAllocateConsistent()
      /app/internal/cli/app_test.go:90 +0x1a4
  testing.tRunner()
      /usr/local/go/src/testing/testing.go:1689 +0x180
  testing.(*T).Run.gowrap1()
      /usr/local/go/src/testing/testing.go:1742 +0x40
==================
--- FAIL: TestCLISpectrumAllocateConsistent (0.05s)
    app_test.go:94: 并发分配退出码应为 0, 实际 6
        错误: model: 频段超额分配: 8/10 轮出现不一致, 累计丢失授权记录 392 条
    testing.go:1398: race detected during execution of test
FAIL
FAIL	satnet/internal/cli	0.078s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

定向测试（含 -race）与全量回归在 linux/amd64、linux/arm64 双架构下均通过。
任意并发度下成功申请次数等于授权记录条数，不出现丢失。
每个频段的已授权带宽等于该频段全部授权记录之和，且不超过该频段总带宽。
go test -race ./... 不报任何数据竞争。
串行申请、带宽耗尽时的拒绝与计数、释放授权的既有行为保持不变。
