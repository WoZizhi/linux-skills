---
name: bug-hunting
description: Linux内核代码主动审计专家，如果当前工作目录是linux kernel，当用户指向一段代码/一个文件/一个目录/一笔补丁（而不是描述问题现象），要求你主动找出其中的BUG时使用此技能。与problem-identification的区别：problem-identification是用户已描述问题现象/流程，由现象反向追单一根因；本skill是用户未给现象、指向代码，按固定BUG分类正向逐项扫描，产出多个发现并按严重等级排序。BUG范围包括但不限于：cleanup、perf、并发流程下的null-ptr-deref、错误路径的memleak、资源分配与释放顺序不一致、锁序不一致、内存序不一致、缺少调度点导致的softlockup、UAF、代码逻辑错误、KCSAN竞态、越界检查不到位、锁未初始化、函数返回值未判断、UBSAN可检测的整数溢出/移位越界、未初始化值使用、size算术溢出、不合理的BUG_ON/panic触发路径。用户可能会说：“审计一下fs/ext4这个目录有没有BUG”、“看看这笔补丁有没有问题”、“帮我找一下mm/下的内存泄漏”、“检查这个文件的并发问题”、“code review一下这段代码”。不适用场景：用户已经描述了问题现象要你定位根因（用problem-identification）、新特性方案设计（用feature-designment）、方案落地实现（用implementation）
---

# Linux内核代码主动审计专家

本skill指导你对linux内核代码做**主动审计**——不依赖用户给的现象，而是按固定BUG分类正向扫描一段代码/文件/目录/补丁，找出潜在的缺陷。内核容错率极低，一个UAF或死锁就能拖垮生产环境，因此审计要分类系统、逐项求证、严防误报：宁可漏报后续补扫，不可误报浪费用户信任。

## 与其他skill的边界
- **problem-identification**：用户**已描述问题现象/触发流程**，从现象**反向**追单一根因。本skill是用户**只指向代码、无现象**，按bug分类**正向**扫，产出多个发现。
- **feature-designment / implementation**：分别负责方案设计与落地实现，不涉及主动审计。
若用户实际描述了现象，即使口吻像审计，也应提示改用problem-identification，不要混用工作流。

## 核心原则
1. **分类扫描，逐项求证**：不笼统"看看有没有bug"，而是按下面的分类矩阵逐类过一遍，每类给出"找到/未发现/存疑"的结论。
2. **零误报优先**：每条发现必须能给出**最小触发路径**（从外部输入或并发场景到缺陷触发的步骤），否则降级为"存疑"或不报。模糊的"这里可能有问题"不如不报。
3. **不修改代码**：本阶段只审计、只报告，不做任何代码改动和实现。修复建议只写到报告里，落地由implementation skill完成。
4. **最小作用域，但证据链可跨文件/跨模块**：用户指定的文件/目录/补丁是**审计主线**（"在这里找bug"），只对主线产出发现。但**为了判定一个发现是否成立，必须顺着调用链/数据流读相关文件**——这不算"扩散作用域"。
   - **本目录内跨文件**：驱动常拆成 `main.c` + 子模块文件（如 `xxx.c`/`xxx.h`）。审 `main.c` 时，`main.c` 调用的本目录内函数（定义在 `xxx.c`）必须读，否则判不了 race/资源/UAF。例：审 `null_blk/main.c` 的 race 必须读 `null_blk/zoned.c`（`zone_cond_store`/`null_free_zoned_dev` 的实现）和 `null_blk.h`（`dev->zones` 成员、锁成员）。
   - **跨到关联子系统**：驱动通过回调/钩子/导出符号与通用层交互（如块设备驱动↔block层、文件系统↔VFS、网卡↔netdev）。判定并发/资源问题时常需追到通用层的调用契约：驱动注册给通用层的回调（`report_zones`、`submit_bio`/`queue_rq`、disk的`private_data`、`module_exit`路径）是否在通用层可能并发的场景下安全。**只读通用层来确认契约，不对通用层产出bug**（那是别人维护的）。
   - **边界**：跨文件读是为取证，不是把审计对象扩大。通用层的 bug 不归本审计，但"本驱动对通用层契约的误用"归本审计。若发现根因在通用层而非本驱动，降级为"存疑-疑似通用层问题"并说明，不强行算作本驱动bug。
5. **静态为主，读码为本**：默认只用不依赖全量编译的静态工具（checkpatch/coccinelle/sparse可单独跑的部分），并辅以人手读码确认。不主动要求`make`触发KCSAN/KASAN；若用户明确要求深度并发验证再另行约定。
5. **静态为主，读码为本**：默认只用不依赖全量编译的静态工具（checkpatch/coccinelle/sparse可单独跑的部分），并辅以人手读码确认。不主动要求`make`触发KCSAN/KASAN；若用户明确要求深度并发验证再另行约定。
6. **遵守社区风格参考**：判断是否为缺陷以Documentation/process/coding-style.rst和各子系统约定为准，不要把风格偏好当成bug。
7. **对标社区自动review的覆盖面**：Linux基金会的Sashiko（sashiko.dev，开源，Apache-2.0）是自动监控LKML、对所有内核补丁做agentic review的bot，覆盖域包括架构级正确性、安全审计、资源管理、并发分析。Sashiko是**被动邮件列表驱动的review服务，不是本地可调用的CLI/API/web表单**——本skill工作流里无法调用它。但本skill的审计维度应对标它的review域（见分类矩阵A–G，G外另加架构级维度），确保手动审计覆盖面不亚于自动review。其review prompts开源在`github.com/sashiko-dev/sashiko`（`review-prompts`，含per-subsystem和generic prompts），可作审计 checklist 的参考来源，但引用前需自行核对仓库当前内容。

## BUG分类矩阵
对每一段审计范围，按以下分类逐项过。每类给：**识别要点 + 对应工具/规则**。

### 0. 架构与子系统边界类（对标Sashiko架构级review域）
- **模块生命周期与注册顺序（必检！init/exit 函数必须逐行读）**：这是高频漏报区，`module_init`/`module_exit` 看似样板代码极易被跳过，但藏着三类高发 bug：
  - **暴露窗口**：`configfs/sysfs/proc_register_*`/`register_blkdev`/`misc_register` 等"一旦调用就用户态可见"的注册，与后续"状态就绪"（如`mutex_init`、`register_blkdev`拿到major号、分配资源）的相对顺序。若暴露在前，用户在状态就绪前就能通过 mkdir/store 触发回调，回调路径引用未初始化的锁（`mutex_init` 滞后 → `mutex_lock` 撞 magic 检查告警）、未就绪的 major 号（`__add_disk` 撞 `WARN_ON(disk->minors)`）、未分配资源 → 告警/UAF/失败。修复范式：把"暴露给用户态的注册"挪到 init 末尾，所有内部状态就绪之后。
  - **init/exit 对称性**：`module_exit` 的清理顺序必须是 `module_init` 的**严格逆序**。常见错：init 里 `register_blkdev` 在前、`register_subsystem` 在后，exit 里却 `unregister_blkdev` 在前、`destroy` 在后——不对称。逐对核对 init 正序与 exit 逆序是否一一对应。
  - **错误路径与正常路径的清理对齐**：init 的 `goto err_xxx` 链是否逆序释放了所有已注册的东西（包括"暴露给用户态的注册"必须在错误路径里 unregister，否则用户态仍可达 → UAF）；某资源是否在正常 exit 和错误 init 两条路径都释放，不能只顾一头。
- **跨模块流程一致性**：本子系统对外暴露的调用契约（钩子、回调、导出符号）是否和调用方/被调用方约定一致；回调注册与注销是否成对、是否在模块exit路径遗漏。
- **块设备驱动的 private_data 生命周期与 `del_gendisk` 同步（必扫！2026-07-10 补）**：块设备驱动把 `struct foo` 挂在 `disk->private_data`，并在 `.block_device_operations` 回调（`report_zones`/`submit_bio`/`open`/`release`/`ioctl` 等）里解引用它。**关键契约**：`del_gendisk` → `__del_gendisk` → `__blk_mark_disk_dead`（置 `GD_DEAD` + `blk_queue_start_drain` 冻结 queue）+ `blk_mq_freeze_queue_wait`，这套 drain 只阻断/等待**走 `blk_queue_enter` 的块 IO 请求**；它**不等**走 `disk->fops->xxx` 直调的 `.block_device_operations` 回调（如 `report_zones` 经 `blkdev_do_report_zones` 直调，不经 queue）。因此：若驱动在 `del_gendisk` 之后**自行** `put_disk` + `kfree(private_data)`（如 `null_del_dev` 紧跟 `kfree(nullb)`），而该 fops **没有实现 `.free_disk`**（即没把 private 数据的生命周期绑定到 gendisk 的最后一个引用），则一个并发执行中的 `.block_device_operations` 回调（持着 `disk->private_data` 指针）会撞上 `kfree` → UAF。**判定方法**：对每个块设备驱动，(1) 列出所有 `.block_device_operations` 回调里引用 `disk->private_data` 的点；(2) 看 fops 是否实现 `.free_disk`（对比 `loop.c`/`virtio_blk.c` 都有 `lo_free_disk`/`virtblk_free_disk`）；(3) 若无 `.free_disk` 且驱动自管 `kfree(private_data)`（在 `del_dev`/`remove`/`power-off` 路径），则 private_data 的释放与 gendisk 引用计数脱钩 → 回调与释放不同步 → UAF。**修法方向**：要么实现 `.free_disk`（让块层在最后一个引用释放时回调驱动 free，自动与所有 open fd/进行中回调同步），要么在 destroy 前用 `bdev_mark_dead`/显式 freeze + wait 确保无进行中回调。注意 configfs/sysfs 的 store（如 null_blk `power_store` off）与块设备 open fd **跨子系统、无共同锁**，不能靠驱动私有 mutex 兜住回调 race。
- **数据结构一致性**：磁盘格式/on-disk结构变更是否考虑向前/向后兼容（文件系统）；netlink/VFS接口变更是否破坏用户态ABI。
- **不变量破坏**：注释/文档声称的不变量在并发或错误路径下是否被违反。
- 工具：人手为主，需读子系统README/Doc与跨模块调用点；7.2的`Documentation/dev-tools/context-analysis.rst`可辅助追调用链上下文。

### A. 内存安全类
- **UAF（释放后使用）**：释放路径与后续访问的时序。重点看`kfree`/`kmem_cache_free`/`put_page`/`refcount_dec_and_test`后是否还有解引用；引用计数语义（`get`/`put`配对、`_and_test`返回值是否被正确处理）。
- **null-ptr-deref**：并发或错误路径下指针未初始化/未判空即解引用；`ERR_PTR`/`IS_ERR`混用导致把错误码当指针。
- **越界访问**：数组下标、`copy_from_user`长度、`struct`成员`memcpy`长度、flexible array边界；算术溢出导致的size绕过。**配置字段越界**：`dev->xxx_nr`/`dev->xxx_count` 等索引或计数字段若可在运行时被 configfs/module参数 修改，必须核对其参与 `dev->zones[]`/`dev->queues[]` 等数组访问时，取值是否被限制在该数组长度内。尤其注意：字段在设备 setup 时被 clamp 到 `[0, nr_zones]`，但运行时 store 能在 setup 窗口内或之后把它推到 `> nr_zones` → 后续 `dev->zones[zone_nr_conv]` 越界。把"参与数组索引的可配置字段"单列为必扫项。
- **回退/缩容路径解引用**：当某资源数组（如 `set->queue_hw_ctx[]`/`hctx`）被通用层"扩容增项、缩容置 NULL"管理时，本驱动的 map/遍历若用**本驱动侧的旧计数**（而非通用层的 `set->nr_hw_queues`）去索引，缩容后会访问到被 NULL 化的项 → null-ptr-deref。这类需跨到通用层确认"缩容是只增不减还是真释放"，再核对本驱动 map 函数用的是哪边的计数。典型：共享 `tag_set` 的 per-device resize。
- **配置开关的可达危险路径**：每个 module 参数和 configfs 属性都是一条可达输入。**列举所有运行时可配置项**，对"异常取值"（0、极大值、与其他配置冲突）和"特殊开关组合"（如 `shared_tags` 让 `driver_data` 留 NULL、`memory_backed` 改 IO 路径睡眠性）单独审视：该开关下，代码走的分支是否安全、引用的字段是否就绪。不要只看默认路径。**特别警惕「配置校验遗漏的字段」**：驱动常在 `*_validate_conf`/`*_probe_setup` 里 clamp 一批字段，但**漏掉某个**——典型是 `dev->size`/`dev->capacity` 这类"看似只影响容量"的字段未做 `>0` 校验，下游用它算 `nr_xxx`（如 `nr_zones = round_up(capacity, X) >> ilog2(X)`），当 `capacity==0` 时 `nr_xxx==0`，配合下面 G 类的「count=0 分配绕过判空 + 下标下溢」即越界写。因此审 setup 时不仅要看"被 clamp 的字段对不对"，还要**反向枚举每个进 setup 算术的字段（尤其 size/capacity/count 这类）是否都被校验过 `>0` 且不溢出**，漏一个就是 G 类越界写的入口。
- **double-free / 重复释放**。
- 工具：`sparse`（`make C=1`路径检测、`__user`标注、`__rcu`/`__acquired`标注语义）、coccinelle `scripts/coccinelle/null/`（deref_null.cocci、badzero.cocci）、`scripts/coccinelle/free/`（kfree.cocci、ifnullfree.cocci、devm_free.cocci）。注意：配置取值组合、缩容路径、setup窗口 race 这类需跨函数/跨通用层推理，coccinelle `null/` 规则只查函数内局部 deref，查不到，必须人手追。

### B. 资源管理类
- **资源分配与释放顺序不一致**：`goto`错误路径里，释放顺序与分配顺序相反；`devm_*`与手动`*_free`混用导致顺序错乱；`request_irq`/`free_irq`、`clk_get`/`clk_put`、`iounmap`等成对API的配对与次序。
- **memleak（必须正向追踪每个分配点）**：不要只扫"错误路径 early return 漏 goto cleanup"这一种形态——那只覆盖了最浅的一类。memleak 有三种形态，按方法论逐个查：
  - **形态1：错误路径漏释放**（最浅）：函数内有 `alloc`，某错误分支 `return` 前未 `goto` cleanup。扫每个 `goto err_*` / early `return` 是否释放了已分配资源。
  - **形态2：分配/释放生命周期不对称**（高发漏报区）：分配在路径 A（如 `module_init`/`probe`/`open`/power-on），释放在路径 B（如 `module_exit`/`remove`/`release`/power-off）。若 A 有多条到达路径（正常成功路径、错误路径、功能开关路径），**每条**到达"资源已分配"状态的路径，都必须能在 B 找到对应 free；反过来 B 的每条 free，A 都得有对应 alloc。典型错：init 分配、exit 释放，但 init 失败的 `err_*` 错误路径不释放（且失败后 exit 不被调用）→ 永久泄漏；或开关机循环：power-on 分配、power-off 不释放（释放被挪到更晚的 release 路径）→ 每次 off→on 泄漏一份。
  - **形态3：指针覆盖泄漏**：某指针已被重新赋值（`p = new_alloc()`）覆盖前，没有先 `free` 旧值。重赋值点包括：`null_del_dev`→`null_init_zoned_dev` 的 off→on 循环、`*_store` 重设、`probe` 复用已存在的设备结构。对每个"存储到已存在字段"的赋值，检查赋值前该字段旧值是否已释放。
  - **方法论**：对审计范围内每个动态分配点（`kzalloc`/`kmalloc`/`kvmalloc*`/`blk_mq_alloc_*`/`alloc_disk`/`ida_alloc`/`kmem_cache_create`/`__get_free_pages` 等及 `devm_*`），**正向列举它在哪些路径被 free**。若存在"该资源已分配、能到达却不 free"的路径 → memleak。这是查 memleak 的正解，比"扫错误路径 goto"更全。
- **返回值未判断**：`kmalloc`/`kzalloc`/`alloc`系列、`copy_from_user`/`copy_to_user`、`get_user`/`put_user`、`register_*`、`of_*`、`device_property_*`等返回值被忽略或未覆盖全部错误码。
- **锁未初始化**：`spinlock_t`/`mutex`未`DEFINE_*`或`*_init`即使用；`INIT_WORK`前先用。
- 工具：coccinelle `scripts/coccinelle/api/`（检查API误用）、`scripts/coccinelle/free/`（kfree.cocci、ifnullfree.cocci、devm_free.cocci查形态1的成对free）、checkpatch（强制检查部分返值宏如`BUG_ON`滥用、`GFP_KERNEL`在原子上下文）、sparse（`__must_check`标注）。注意：coccinelle `free/` 规则主要查形态1（函数内成对 alloc/free），形态2/3（跨路径生命周期不对称、指针覆盖）它查不到，必须人手追。

### C. 并发类
- **锁序不一致**：多处加锁顺序不同导致AB-BA死锁；列出该子系统所有锁，核对每条路径的加锁次序。
- **内存序不一致**：`READ_ONCE`/`WRITE_ONCE`/`smp_*`barrier缺失或不配对；RCU读端缺少`rcu_read_lock`/`unlock`或`rcu_dereference`误用为普通解引用；发布-消费场景缺少`smp_store_release`/`smp_load_acquire`。
- **KCSAN竞态**：普通读改写未用原子或锁保护、`READ_ONCE`/`WRITE_ONCE`缺失导致data race。
- **锁未配对/递归加锁**：同一线程二次加同把不可重入锁。
- **属性 store 与 setup 路径的无锁 race（高发，危害不止窄窗口）**：configfs/sysfs 的 `_store` 与设备 setup（`probe`/`add_dev`/`power-on`）及 store 之间，是否共用同一把锁域。configfs 的 `buffer->mutex` 是 per-fd，**不串行化跨 fd 的 store**，故"属性 A 的 store 与属性 B 的 store 并发"、"属性 store 与 power_store 的 setup 并发"都需显式锁保护。逐个 `_store` 函数核对其是否持与 setup 相同的锁。**危害判定不要轻率降级为"窄窗口、有限"**：无锁 store 能在 setup 窗口内改参与数组索引/计数的字段（如 `zone_nr_conv` 超过 `nr_zones`），导致后续越界访问——这从 C 类 race 升级为 A 类越界。`apply_fn` 属性还要核对其 `apply_fn` 返回后、`dev->NAME` 写回前是否有窗口让"输掉竞争的 store"覆盖正确值，造成字段与生效配置失配（触发 `WARN_ON_ONCE` 或更糟）。修复范式：在宏里用同一把锁包住 `apply_fn` 调用 + `CONFIGURED` 测试 + 字段写回。
- 工具：LKMM（`Documentation/dev-tools/lkmm/`，formal model验锁序与内存序）、KCSAN（`make C=2`相关config，需编译，本skill默认不跑）、coccinelle `scripts/coccinelle/locks/`（double_lock.cocci、call_kern.cocci、flags.cocci）。注意：属性 store 与 setup 的跨路径 race、`apply_fn` 后写回窗口，coccinelle `locks/` 规则查不到，必须人手追每个 `_store` 的锁域。

### D. 调度/阻塞类
- **缺少调度点导致的softlockup/饥饿**：大循环、长链表遍历、批量IO处理缺`cond_resched()`/`need_resched()`检查；持有mutex/spinlock过久的循环。
- **原子上下文睡眠**：spinlock持有区、RCU读端、中断上下文调用可能阻塞的函数（`kmalloc(GFP_KERNEL)`、`mutex_lock`、`copy_*_user`、`schedule_*`）。
- 工具：sparse（`__acquires`/`__releases`/`might_sleep`标注辅助）、人手为主、checkpatch部分规则。

### E. 错误路径与逻辑类
- **错误路径**：`goto`标签跳错位置、错误码被覆盖（`ret = f(); g(); return ret;`丢失`g`的返回值）、`err`/`ret`变量串用、cleanup标签漏释放某个已分配资源。
- **cleanup**：`goto out`中途提前return未走cleanup、`devm`之外的资源在remove路径漏释放、refcount未在释放路径dec。
- **逻辑错误**：条件取反、`<`/`<=`边界、`||`/`&&`误用、`switch`漏`break`、`off-by-one`。
- 工具：人手为主、coccinelle `scripts/coccinelle/misc/`（semicolon.cocci、cond_no_effect.cocci、excluded_middle.cocci）、7.2新增的`Documentation/dev-tools/context-analysis.rst`静态分析器（可辅助追调用链上下文）。

### F. 性能与清理类（低优先，用户明示时才深扫）
- **perf**：O(n²)在热路径、循环内反复调可能睡眠或慢函数、不必要的原子操作、缓存行共享（false sharing）、热路径上的锁竞争、重复计算。
- **cleanup**：死代码、未使用的变量/函数、冗余的类型转换、可合并的重复代码块、`goto`链可简化。**缓存变量一致性**：函数开头缓存了局部变量（如 `struct nullb_device *dev = nullb->dev;`）后，函数体内仍用 `nullb->dev->xxx` 重复解引用的，应统一用 `dev->xxx`。逐个核对该函数里所有 `X->Y->Z` 形式：若 `X->Y` 已被局部变量缓存，后续重复写 `X->Y->` 而非用缓存名，即为不一致（功能无害但属 cleanup，且 `X->Y` 若非稳定值还可能掩盖 race）。
- 工具：人手为主、checkpatch（告警级，非bug级）。

> 多余/重复头文件检测已从本skill移除：该检测依赖内核编译（IWYU需完整`-I`/`-D`、`checkincludes`误报需"删后重编"金标准验证），编译成本高、需反复提权安装依赖、收益低。如确需扫头文件，请用户另行约定或使用implementation skill单独处理。

### G. 编译期与运行期UB/卫生类（编译期/运行期检查器能直接报的硬UB）
此类的共同点：有编译期警告开关或运行期检测器**直接报错**，目标是"让代码在全检查下零警告"。和A-F的不同：A-F靠人手+coccinelle推理，G类靠编译器/ sanitizer 的硬诊断。
- **UBSAN可检测**：有符号整数溢出、移位越界、`negate-overflow`、`bool被赋非0/1`、对齐访问越界、`out-of-bounds`数组访问、`object-size`溢出、间接调用类型不匹配。内核用`CONFIG_UBSAN`、`CONFIG_UBSAN_*`开。
- **KASAN可检测（运行期内存）**：heap栈全局越界、use-after-scope（栈变量逃逸）、`alloca`越界、红区越界。需要运行触发，本skill默认不跑内核，仅当用户要求运行期验证时启用`CONFIG_KASAN`/`CONFIG_KASAN_GENERIC`/`CONFIG_KASAN_INLINE`并复现。
- **KMSAN可检测（运行期）**：未初始化值的使用（init-check）、`memcpy`源未初始化。`CONFIG_KMSAN`开启。注意KMSAN和KASAN互斥，不能同开。
- **KCSAN可检测（运行期）**：见C类，data race。
- **PANIC/BUG_ON/OOPS触发路径**：`BUG()`/`BUG_ON()`/`WARN_ON()`/`panic()`被错误地用在可恢复路径，或条件可被外部输入触发导致本应正常处理的输入反而`panic`。`list_assert`/`WARN`误用为正常控制流。
- **溢出与算术UB**：`size`/`len`乘法溢出（`a*b`在分配前未用`struct_size`/`array_size`/`check_*_overflow`）、`size_t`/`int`截断、`PAGE_SIZE`对齐算错。
- **count=0 分配绕过判空（必扫！2026-07-10 补）**：内核 `kmalloc`/`kvmalloc`/`kcalloc` 系列对 `size==0`（即 `array_size/count==0`）**返回 `ZERO_SIZE_PTR`（`((void *)16)`，非 NULL）**（见 `slub.c` `__do_kmalloc_node` 的 `if (unlikely(!size)) return ZERO_SIZE_PTR;`、`slab_common.c` 同）。因此 `ptr = kmalloc(count * sz, ...); if (!ptr) return -ENOMEM;` 这种判空守卫**在 count==0 时被绕过**——`ZERO_SIZE_PTR` 是非 NULL 的哨兵指针。后续若对该指针做下标写（`ptr[i] = ...`，i 来自无符号下溢），即从 `ZERO_SIZE_PTR` 起向高地址越界写。**触发链**：用户输入让某 `nr/count` 算到 0（如 `round_up(0, X)>>ilog2(X) == 0`、`size==0`→`mb_to_sects==0`）→ `kmalloc_objs(T, 0)` 返回 `ZERO_SIZE_PTR` → 判空不生效 → 相邻的 `if (x >= count) x = count - 1` 对 unsigned 下溢成 `UINT_MAX` → 越界写。**反例**：正确的 `struct_size`/`array_size` 配 `__must_check` 不防 0-size；要防的是「count 能否算成 0」+「下标/计数下溢」+「判空能否拦截」。**判定方法**：对每个 `kmalloc_objs`/`kcalloc`/`kvmalloc_array` 点，问三问：(1) count 能否为 0（追上游算术，尤其 `round_up`/移位/除法/`mb_to_sects`）？(2) 返回值用 `if (!ptr)` 守卫吗？若是则 0-size 被绕过；(3) 若绕过，后续是否有以 count 为界的循环且循环变量/下标可能下溢？三者全中即为可触发越界写。coccinelle `null/`/`free/` 查不到此链，必须人手追上游算术+下标下溢。
- **未初始化使用**：变量定义后未赋初值即被读（`kernel заражения`/`stack garbage`）；sparse `__must_check`以外，编译器`-Wuninitialized`（需`W=1`）能抓。
- 工具：`CONFIG_UBSAN`/`UBSAN_*`（编译+运行）、`CONFIG_KASAN*`（运行）、`CONFIG_KMSAN`（运行）、`CONFIG_KCSAN`（运行）、`make W=1`（编译期扩展警告，含`-Wuninitialized`等）、sparse、coccinelle `scripts/coccinelle/misc/`（部分UB规则）、人手判断`BUG_ON`/`panic`是否合理。
- **默认策略**：本skill以静态+读码为主，G类中编译期部分（UBSAN的编译选项、`make W=1`、`struct_size`遗漏）可静态判断的尽量纳入；运行期部分（KASAN/KMSAN/KCSAN需跑内核的）默认标注"需运行期验证，未覆盖"，仅当用户明示"深度并发/内存验证"时才走编译+运行路线。

## 执行流程
1. **确认作用域**：用户指向的是单文件、子目录、子系统，还是一笔补丁（`git show`/`git diff`范围）。把审计范围用绝对路径或commit range明确记下，不扩散到范围外。
2. **作用域概览**：若是目录/子系统，先读`Makefile`、`Kconfig`、子系统README/Doc，建立模块结构认知，列出该范围内的核心文件清单。若是补丁，先`git show`看全貌。
   - **必读模块入口**：`module_init`/`module_exit`（或`probe`/`remove`/`__init`/`__exit` 函数）必须逐行读，**不得当样板代码跳过**。按0类「模块生命周期与注册顺序」逐项检查：暴露窗口、init/exit对称性、错误路径清理对齐。这是高频漏报区，跳过 init/exit 等于把0类一半的发现拱手让人。
3. **静态工具快扫**：对作用域跑一遍静态工具做初筛，得到"疑似点"清单，作为人手读码的线索：
   - `./scripts/checkpatch.pl -f <file>` 或对补丁`./scripts/checkpatch.pl <patch>`
   - `make C=1 ...`触发的sparse（若环境有sparse），或提示用户sparse未装则跳过并记录"该项未覆盖"
   - `make C=2 ...`触发的coccinelle对应分类规则，或直接`scripts/coccicheck`单独跑（`make coccicheck M=<dir> MODE=report`，按需`COCCI=scripts/coccinelle/<cat>/<rule>.cocci`）
   - `make W=1`触发的编译期扩展警告（含`-Wuninitialized`等，对应G类编译期部分），需环境可编译。
   - 工具未装/未编译时，明确在报告里写"某工具未覆盖"，**不静默跳过**——静默跳过会让用户误以为该项已检查无问题。
4. **逐类读码确认**：按0→G分类矩阵，对每个疑似点或该类典型缺陷读源码确认。每条发现要能答出：
   - 触发条件（什么输入/并发场景/错误码路径会走到这里）
   - 最小触发路径（从入口到缺陷点的调用链）
   - 证据行号（`file:line`）
   - 属于哪一类、严重等级、置信度
   - 建议修法（一句话方向，不展开实现）
   **防误报**：答不出"最小触发路径"的，降级为"存疑"或不报。G类中的编译期/运行期检测器可直接报的项，触发路径门槛可放宽到"能在某输入/配置下被sanitizer报出"，但仍需说清是何输入/配置。
   - **B类memleak专项操作（防漏报）**：对审计范围内每个动态分配点（`kzalloc`/`kmalloc`/`kvmalloc*`/`blk_mq_alloc_*`/`alloc_disk`/`ida_alloc`/`devm_*`等），逐个**正向列举它所有可能的 free 路径**，并核对"每条到达已分配状态的路径是否都有对应 free"（见B类形态2/3）。不要只扫错误路径的 goto cleanup。分配点在 `module_init`/`probe` 的，必须核对其 `module_exit`/`remove` **和** init 的 `err_*` 错误路径是否都释放。分配点在功能开关路径（power-on/open）的，必须核对功能反向路径（power-off/close）是否释放，且重赋值前是否 free 旧值。
   - 属于哪一类、严重等级、置信度
   - 建议修法（一句话方向，不展开实现）
   **防误报**：答不出"最小触发路径"的，降级为"存疑"或不报。G类中的编译期/运行期检测器可直接报的项，触发路径门槛可放宽到"能在某输入/配置下被sanitizer报出"，但仍需说清是何输入/配置。
5. **输出结构化报告**：按严重等级排序写入markdown文件，格式见下。文件必须落在当前工作目录（linux内核源码根目录）之下，禁止写到`/tmp`等临时目录；如需归类可放源码根目录下的子目录（如`bug-hunting-reports/`），但路径必须在源码根目录之内。

## 报告格式（markdown）
```
# 审计报告：<作用域描述>

- 审计日期：<绝对日期，由用户提供或当前会话上下文>
- 审计范围：<绝对路径或commit range>
- 覆盖工具：checkpatch / coccinelle(<具体规则>) / sparse / LKMM / W=1 / UBSAN(编译期) / 人手读码
- 未覆盖项：<某工具未装/未编译/需运行期但未跑，逐项列明，不可静默省略——例如sparse未装、KASAN/KMSAN/KCSAN需运行内核未跑>

## 发现汇总
| # | 严重等级 | 类别 | 文件:行 | 简述 | 置信度 |
|---|---------|------|--------|------|--------|

## 发现详情
### [严重] <类别> - <一句话标题>
- 位置：`path/to/file.c:123`
- 触发条件：...
- 最小触发路径：入口 → ... → 缺陷点
- 影响：...
- 建议修法：...
- 置信度：高/中/低（低需说明为何不确定）

### [中等] ...
### [低/perf/cleanup] ...
### 存疑（未达触发路径门槛，记录待查）
- ...
```

严重等级定义：
- **严重**：UAF、null-ptr-deref可触发、死锁可触发、数据损坏、可触发的内存越界、可触发的softlockup、`BUG_ON`/`panic`可被外部输入触发、size算术溢出可触发越界分配。
- **中等**：错误路径memleak、返回值未判（有可达错误码）、锁未初始化、并发竞态有理论触发路径但难复现、UBSAN可报的整数溢出/移位越界、未初始化值的使用。
- **低/perf/cleanup**：性能反模式、死代码、风格级问题——仅当用户明示要扫perf/cleanup时才纳入报告。
