# SGLang：HDD 接入方案与改动范围

> 接口可行性 · 2026-11-16 固定版本源码核查

最小原型有明确接入点：将 HiCache 的 L3 文件后端目录放到 HDD 文件系统。这个判断来自后端路径与普通文件 I/O 的代码，并不代表 HDD 已通过性能验证。

本节把“最低级”理解为存储层级的最底层 L3，而不是修改 attention kernel。固定提交 `c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab`；未在 GPU + HDD 环境运行，模型兼容性和服务目标仍需验证。 [SGD][SGF]

| 层级 | 作用 | HDD 接入关系 |
|---|---|---|
| L1 · GPU HBM | 实际推理使用的 KV | 保留现有 GPU 计算路径 |
| L2 · CPU DRAM | 每个实例私有的 host pool | 接收 L3 预取，随后回填 GPU |
| L3 · 文件 / 远端后端 | 按页存储、匹配、恢复 KV | 本地 HDD 挂载点可承载文件；共享路径不自动保证并发一致性 |

### 源码依据

HiCacheFile 构造函数优先读取 SGLANG_HICACHE_FILE_BACKEND_STORAGE_DIR，其次 extra_config.file_storage_path，最后默认 /tmp/hicache。读取经 open(..., buffering=0)、文件大小检查和 readinto；写入先生成临时文件，再 os.replace 发布。这里没有要求存储设备必须是 SSD。

~~~python
# 源码出处（节选） https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/hicache_storage.py#L896-L900
self.file_path = (
    envs.SGLANG_HICACHE_FILE_BACKEND_STORAGE_DIR.get()
    or (storage_config.extra_config or {}).get("file_storage_path")
    or file_path
)

# 源码出处（节选） https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/hicache_storage.py#L1053-L1059
expected = target_location.numel() * target_location.element_size()
with open(tensor_path, "rb", buffering=0) as f:
    if os.fstat(f.fileno()).st_size != expected:
        raise IOError(f"Incorrect object size for {suffixed}")
    buf = memoryview(target_location.view(torch.uint8).contiguous().numpy())
    if f.readinto(buf) != expected:
        raise IOError(f"Short read for {suffixed}")

# 源码出处（节选） https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/hicache_storage.py#L1117-L1120
tmp_path = os.path.join(self.file_path, f".{uuid.uuid4().hex}.tmp")
value.contiguous().view(dtype=torch.uint8).numpy().tofile(tmp_path)
os.replace(tmp_path, tensor_path)
self._evictor.commit(suffixed)
~~~

### 可以直接配置的原型

~~~bash
# 出处（据此整理） [SGD] https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/docs/docs/advanced_features/hicache_design.mdx
# 出处（据此整理） [SGF] https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/hicache_storage.py
# 示例；尚未执行。模型、GPU 资源与 page size 需符合目标版本支持范围。
# /mnt/hdd/hicache 必须实际位于 HDD，而不是系统 SSD 或 tmpfs。
SGLANG_HICACHE_FILE_BACKEND_STORAGE_DIR=/mnt/hdd/hicache \
python -m sglang.launch_server --model-path <compatible-model> \
  --enable-hierarchical-cache \
  --hicache-storage-backend file \
  --hicache-storage-prefetch-policy timeout
~~~

这条路径用来验证 KV 写入、冷恢复和重算回退，不应直接作为生产部署配置。L2 容量、页大小和 timeout 必须按模型与负载设置。文档中的默认 timeout 为 min(30s, 2s + 0.1s × 待取 tokens / 1024)，不等于业务可接受的等待预算。

### 接口可行、性能可行、生产可行应分别判断

| 判断层次 | 目前证据 | 尚缺验证 |
|---|---|---|
| 能否接入 | 普通文件路径后端与 HiCache L3 接口已存在 | 实际挂载介质、模型 / host pool / TP 支持、读写成功 |
| 能否比重算更合算 | 支持预取终止与部分已取前缀继续计算 | 全冷恢复耗时、有效命中、P99 TTFT、GPU 等待 |
| 能否长期运行 | 当前文件实现含容量 / LRU 相关处理与原子发布 | 掉电语义、后台写回稳定性、共享访问、压力和故障测试 |

### 接入路线与改动范围

| 路线 | 可利用的能力 | 新增工作与边界 |
|---|---|---|
| A · file / fast_file → HDD 文件系统 | 现成 L3 文件接口；fast_file 文档列出并行读取、原子发布与后台 LRU | 先建立正确性基线；并发读不等于机械盘更快，需实测 |
| B · L3 内部 SSD + HDD 分层 | SGLang 仍看到一个 L3 后端；SSD 可承接热数据与写入 | 需由文件系统 / 块层缓存或自定义后端提供分层；HiCache 不会自动增加一个独立 SSD 层 |
| C · dynamic 自定义后端 | 文档支持 backend_name / module_path / class_name | 实现读取、写入、存在性及适用的 v2 host-pool 接口；增加合并 I/O、索引、准入和队列管理 |

[SGD][SGF][SGFAST]

### HDD 特有瓶颈

file 的 legacy batch_get 是逐页循环，v2 也按 pool / page 调用读取。每个对象独立命名不保证物理连续，扩大 page size 也不等于把多个对象自动合成一次顺序 I/O。HDD 寻道、元数据查询和多 rank 并发会使小对象随机访问变贵。

fast_file 源码明确列出 readv / writev、默认 1 个 read_workers、按模型 / 布局的 namespace，以及 max_size / min_free_space 等容量设置。锁、淘汰计数和元数据缓存为进程本地，因此多进程共享目录仍需单独核查。fast_file 的并行读是可比较的已有路径；HDD 需要按阵列和队列深度调参，不能沿用 SSD 的线程数就声称性能改善。若开发 extent 容器，需要额外解决对象到偏移映射、垃圾回收、崩溃恢复与页级有效性。

### 不要混淆三个“direct / complete”

--hicache-io-backend 的 direct / kernel 指 CPU 与 GPU 间的传输。Python buffering=0 只关闭用户态缓冲，不能证明使用 O_DIRECT 或绕过 OS page cache。os.replace 提供文件名的原子发布；所读 file 写路径未见 fsync，不能据此承诺掉电后持久化完成。

### 恢复与写回控制

文档提供 best_effort、wait_complete 和 timeout 预取策略。wait_complete 可能把 HDD 冷读等待暴露在 TTFT；timeout 可以限制等待，但还要检查停止后在途 I/O、已恢复前缀、未恢复部分重算及各 rank 一致性。文档列出 write_back / write_through / write_through_selective，不能把这些配置名理解成设备侧持久化或 HDD 顺序写保证。

工程候选：按前缀复用价值控制准入、将写回聚合为大块、为前台读保留带宽、设置后台积压上限、在预测恢复比 prefill 更慢时回退重算。这些是需要开发和对照验证的策略，不是本次已实现功能。

## 本章资料

- **[SGD]** [SGLang HiCache 设计文档 · 固定提交 c712459e7f82](https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/docs/docs/advanced_features/hicache_design.mdx)
- **[SGF]** [HiCacheFile / HiCacheStorage 原始源码](https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/hicache_storage.py)
- **[SGFAST]** [HiCacheFastFile 原始源码](https://github.com/sgl-project/sglang/blob/c712459e7f8233302fdf0bdd5d4ddcd22aa4b2ab/python/sglang/srt/mem_cache/storage/fast_file/fast_file_store.py)
