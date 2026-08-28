# UMI KSU 启动稳定性修复计划

## 当前 Git 状态

- 当前分支：`feat/umi-box4magisk-netfilter`
- 当前提交：`cbc4d4b5d64f`
- 当前分支相对 `origin/feat/umi-box4magisk-netfilter`：ahead `0`、behind `0`
- 本地另一个分支 `android15-lineage22-mod` 也与对应远端一致。
- 按本地已有的 `origin/*` 跟踪引用判断，没有只存在于本地、尚未推送到 GitHub 的 commit。
- 保存计划前，工作区有 13 个未提交的 netfilter/litmus 修改，没有 staged 或其他 untracked 文件。这些修改必须保留，不能回退或覆盖。当前唯一新增的 untracked 文件是本计划文件本身。
- 以上比较基于本地缓存的远端引用；开始实施前应再执行一次联网 `git fetch origin` 做最终确认。

## SELinux 主修复

最高风险是 4.19 legacy SELinux 路径：

```text
init second_stage
→ apply_kernelsu_rules()
→ write_lock(policy_rwlock)
→ preempt_enable()
→ GFP_KERNEL / cond_resched()
```

这会在持有自旋式 `rwlock_t` 时允许调度和阻塞分配，其他 CPU 的 SELinux reader 可能持续自旋，最终表现为 PID 1 卡住、Logo 阶段无法继续并被 watchdog 重启。

在 `patches/sukisu-4.19-compat.patch` 中修改构建时拉取的固定 SukiSU core：

1. legacy `apply_kernelsu_rules()` 先通过 `security_read_policy()` 获取策略快照。
2. 在锁外通过 `policydb_read()` 建立私有 policydb 副本。
3. 在私有副本上执行 KSU 规则修改，允许正常使用 `GFP_KERNEL` 和调度。
4. 规则失败时丢弃副本，不安装部分修改的策略。
5. 捕获 policy generation，安装前检查并发策略更新；冲突时重试一次。
6. 只在短暂的 `write_lock_irqsave()` 临界区替换 policydb。
7. 解锁后销毁旧 policydb、刷新 AVC，并在 legacy 路径调用 `susfs_set_batch_sid()`。
8. `handle_sepolicy()` 使用相同的私有副本流程，不再直接修改全局 policydb。
9. 删除 legacy 路径中的 CPU pinning、FIFO 50 调度和手动抢占开关。
10. 没有可用 policy lock 时返回错误，禁止回退到不安全的直接修改路径。
11. 5.10 及以上的 mutex/RCU 实现保持不变。

## Debug 与持久日志

新增独立的 `CONFIG_KSU_BOOT_DEBUG`，不复用会放宽权限的 `CONFIG_KSU_DEBUG`。

调试日志统一使用固定前缀 `ksu_boottrace`，记录以下阶段：

```text
second_stage_enter
policy_clone_begin
policy_clone_done
rules_begin
rules_done
policy_lock_before
policy_lock_acquired
policy_unlock_done
avc_reset_done
sid_done
second_stage_done
```

每条边界日志至少包含 PID、进程名、CPU、`preempt_count`、调度策略和 RT 优先级。锁内只保留少量边界标记，避免大量 printk 改变时序。

将 4 MiB ramoops 分为：

- dmesg crash record：2 MiB
- console：1 MiB
- pmsg：1 MiB

显式启用 `CONFIG_PSTORE_LAST_KMSG=y` 和 `CONFIG_PRINTK_TIME=y`。

debug 构建额外启用：

```text
CONFIG_DEBUG_KERNEL=y
CONFIG_FRAME_POINTER=y
CONFIG_KALLSYMS_ALL=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_DEBUG_PREEMPT=y
CONFIG_PROVE_LOCKING=y
CONFIG_SOFTLOCKUP_DETECTOR=y
CONFIG_DETECT_HUNG_TASK=y
CONFIG_WQ_WATCHDOG=y
CONFIG_PANIC_ON_OOPS=y
CONFIG_BOOTPARAM_SOFTLOCKUP_PANIC=y
CONFIG_BOOTPARAM_HUNG_TASK_PANIC=y
CONFIG_PANIC_TIMEOUT=5
```

第一轮不启用 KASAN，避免大幅改变内存布局和启动时序。

## 构建与验证

`build.sh` 增加可选参数并保持现有命令兼容：

```sh
bash build.sh umi ksu
bash build.sh umi ksu debug
bash build.sh umi ksu debug no-kpm
```

- `debug`：启用诊断配置和 `ksu_boottrace`。
- `no-kpm`：关闭 `CONFIG_KPM` 并跳过 `patch_linux`，用于单变量 A/B。
- AOSP 和 MIUI 两套构建使用相同的 debug/KPM 选项。

验证顺序：

1. 对固定 SukiSU commit 执行 compat patch `apply --check`。
2. 构建 release、debug 两种 UMI 配置并检查最终 `.config`。
3. 检查 `git diff --check`、编译告警和 policydb 生命周期。
4. 设备执行至少 20 至 30 次重启或冷启动。
5. 验证 ReSukiSU 能识别驱动、获取 root、加载 sepolicy。
6. 从 `/sys/fs/pstore/`、`/proc/last_kmsg` 和 `dmesg` 收集日志。
7. 如果仍复现，仅切换 `no-kpm`，每次只改变一个变量。

重点搜索：

```text
ksu_boottrace
BUG: sleeping function called from invalid context
blocked for more than
soft lockup
hung task
watchdog
RCU stall
```

验收条件：完整出现 `second_stage_enter` 至 `sid_done`，没有 atomic sleep、lockup、RCU stall、watchdog 或 policy generation 异常；UMI 能稳定越过 Logo 进入开机动画和 ADB；ReSukiSU 仍能识别并使用 KSU 驱动。

现有 13 个未提交的 netfilter/litmus 修改必须保持原样。

## 子代理执行流程

计划确认后再启动：

1. `coder` 是唯一允许写代码的代理，负责实现、构建和自检。
2. `reviewer` 只读审查 diff、锁语义、内存生命周期、4.19 API 兼容性和验证结果。
3. reviewer 的问题先返回主代理仲裁，再发送给 coder 修复。
4. 修复后 reviewer 重新审查，直到通过或明确说明阻塞原因。
5. 主代理不直接编辑文件，只负责范围控制、冲突仲裁和最终验收。
6. 子代理运行期间约每 5 分钟检查一次状态，不进行高频轮询。
