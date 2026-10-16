# ComfyUI 使用过程记录

记录 ComfyUI 在本机（Windows / CPU 模式）使用过程中遇到的问题与阶段性总结。

**按时间倒序排列，最新记录在最上方。**

## 记录索引

| 日期 | 记录 |
|------|------|
| 2026-10-01 | 日常使用问答（图片目录 / 耗时 / 内存 / 参数 / 配置存储） |
| 2026-09-30 | Z-Image 出图失败：VAELoader 误用 pixel_space |
| 2026-09-30 | 服务运行中崩溃的排查（Z-Image 加载时访问违规） |
| 2026-09-30 | 模型下载失败（Chrome `.crdownload`）的恢复 |
| 2026-09-29 | 独立 Python 环境搭建、依赖安装与服务启动 |

---

## 2026-10-01 · 日常使用问答（图片目录 / 耗时 / 内存 / 参数 / 配置存储）

> 以下问答整理自 2026-09-30 ~ 2026-10-01 的使用过程，属知识性咨询，非故障排查。

### Q1：本地生成的图片放在哪个目录？

默认在 ComfyUI 根目录下的 `output`，本机即 **`d:\ComfyUI\output\`**。依据 [folder_paths.py](../folder_paths.py#L69-L70)：

```python
output_directory = os.path.join(base_path, "output")
temp_directory = os.path.join(base_path, "temp")
```

| 目录 | 路径 | 用途 |
|------|------|------|
| 输出 | `d:\ComfyUI\output\` | `SaveImage` 保存的图片 |
| 输入 | `d:\ComfyUI\input\` | `LoadImage` 读取的图片 |
| 临时 | `d:\ComfyUI\temp\` | 预览图、中间文件 |

文件名由 `SaveImage` 节点的 `filename_prefix`（当前为 `ComfyUI`）+ 5 位序号组成，如 `ComfyUI_00001_.png`。prefix 含斜杠时会自动建子目录。可用启动参数 `--output-directory "D:\AI-output"` 改变位置。

### Q2：两张已生成图片的耗时分别是多少？

| 项目 | ComfyUI_00001_.png | ComfyUI_00002_.png |
|------|--------------------|--------------------|
| 生成时间 | 2026-09-30 18:46:35 | 2026-10-01 20:34:59 |
| 参数 | 512×512，8 步，seed 42 | 完全相同 |
| 采样单步 | 262.27 秒 | 推算约 168 秒 |
| 采样总时长 | 34:58（实测） | 推算约 22.5 分钟 |
| 任务总耗时 | **39:23（实测）** | 约 24–27 分钟（推算） |

- 图 1 有完整日志：`Prompt executed in 00:39:23`，其中净采样 34:58，其余约 4 分 25 秒为模型加载 + 文本编码 + VAE 解码；
- 图 2 的运行日志不可得（服务在外部终端启动、ComfyUI 默认不写日志文件、PNG 元数据只有 `prompt` 和 `workflow` 两个字段、不含耗时），故为推算值；
- 两张图的 PNG 元数据均只包含 `prompt` 与 `workflow`，不含任何耗时信息。

### Q3：每次重启服务后第一次生图都会更慢吗？

**机制上成立**：重启会清空模型缓存，第一次运行必须重新加载模型（文本编码器 7.67 GB + 扩散模型 23.5 GB），本次实测这部分开销约 4 分 25 秒。

但要注意，采样单步从 262 秒降到 168 秒的差异**不能全归因于"第一次"**——模型加载发生在采样之前，不会影响单步速度。更可能的原因是**运行时内存压力**：图 1 那次系统空闲内存只剩约 1.2 GB，部分权重被换出到页面文件，采样时反复缺页换入换出，单步被拖慢。

结论：冷启动必然多付一次加载成本；单步快慢主要取决于内存是否够用（建议运行时保持 20 GB 以上空闲内存）。

### Q4：一次跑多张图会让内存占用翻倍吗？

**不会。** 内存大头是模型权重（约 31 GB），任何情况下都只加载一份。

| 部分 | 大小 | 是否随张数增长 |
|------|------|----------------|
| 模型权重（文本编码器 + 扩散模型） | 7.67 GB + 23.5 GB | 不增长 |
| 采样中间激活 | 几百 MB 量级（512×512） | 随 batch 线性增长 |
| 输出图像 | 约 3 MB/张 | 可忽略 |

| 运行方式 | 模型权重 | 中间激活 | 总体影响 |
|----------|----------|----------|----------|
| batch_size = 2（同批并行） | 1 份 | ×2 | 增加几百 MB ~ 1 GB（估算） |
| 队列连续跑 2 个任务 | 1 份 | 用完即释放 | 基本不变，最省内存 |
| 两套用不同模型的工作流 | 2 份叠加 | — | 显著增加 |

真正需要留意的是 ComfyUI 的结果缓存（`Using RAM pressure cache`）：默认活跃缓存阈值约为系统内存的 10%（本机约 3.2 GB），连续快速跑多张时会累积。

建议：想跑多张优先用队列连续跑，而不是调大 `batch_size`。

### Q5：当前的 batch_size 是多少？

**1**。依据最近一张图的元数据：

```json
[68] EmptySD3LatentImage: {'width': 512, 'height': 512, 'batch_size': 1}
```

注意：在 ComfyUI 中 batch 由 latent 决定（`EmptySD3LatentImage` 节点的 `batch_size` 参数），`KSampler` 本身没有独立的 batch 设置。

### Q6：seed（随机种子）是什么？怎么改？

`seed` 是决定"从哪一团随机噪声开始生成"的数字。扩散模型从纯随机噪声逐步去噪成图像，这个初始噪声由 seed 通过伪随机算法算出：

- seed 相同 + 提示词相同 + 参数相同 → 每次得到完全相同的图；
- seed 不同 → 得到不同的图（提示词不变时画面主题接近，构图与细节会变）。

当前工作流是 `seed = 42` + `control_after_generate: fixed`，所以每次运行都出同一张图。`KSampler` 的 seed 输入框右侧下拉菜单控制每次运行后的行为：

| 选项 | 行为 |
|------|------|
| `fixed` | 运行后不变（当前设置） |
| `increment` | 每次运行后 seed +1 |
| `decrement` | 每次运行后 seed −1 |
| `randomize` | 每次运行前随机取新 seed |

改 seed 不影响耗时（仍是 8 步采样）。另外，若把 `batch_size` 改为 2 而 seed 保持 `fixed`，两张图会完全相同。

### Q7：ComfyUI 的配置是以文件形式存储的吗？

基本是，只有一处用 SQLite 数据库：

| 配置类型 | 存储形式与路径 | 说明 |
|----------|----------------|------|
| 界面设置 | `user\default\comfy.settings.json` | 面板开关、教程状态、模板排序等 |
| 用户数据 | `user\comfyui.db` | SQLite，存资产记录、标签等 |
| 保存的工作流 | `user\default\workflows\` | 见 Q8 |
| 自定义样式 | `user\default\user.css`（可选） | 不存在时前端会报 `ERR_ABORTED /api/userdata/user.css` |
| 模型路径扩展 | `extra_model_paths.yaml`（根目录，需手动创建） | 用于挂载外部模型目录 |
| 启动参数 | 不是文件 | 通过命令行或 `.bat` 脚本传入 |

[user_manager.py](../app/user_manager.py#L44) 中有一条日志印证了存储方式的变化：

```
User settings have been changed to be stored on the server instead of browser storage.
```

即设置已在服务端文件存储，前端通过 `/userdata/{file}` 接口读写（[user_manager.py](../app/user_manager.py#L334-L362)）。备份/迁移时拷贝 `user\` 整个目录即可带上设置、数据库与工作流。

### Q8：保存的生图工作流放在哪个目录？

**`d:\ComfyUI\user\default\workflows\`**。

路径由两段规则组合而成：

1. 前端定义工作流基路径为 `workflows/`（`comfyui_frontend_package` 静态资源中：`class ComfyWorkflow extends ... { static basePath = 'workflows/' }`），通过 `/userdata/workflows/<文件名>.json` 读写；
2. 后端的 userdata 路径规则是 `user/<用户 ID>/<文件>`（[user_manager.py](../app/user_manager.py#L72-L103)），`user` 根目录在 [folder_paths.py](../folder_paths.py#L72) 中定义，默认用户 ID 为 `default`。

需要区分的三种情况：

| 情况 | 存储位置 |
|------|----------|
| 界面里点**保存**（Save）的工作流 | `user\default\workflows\<名字>.json`（服务端文件） |
| **未保存**的临时工作流 | 浏览器本地存储（服务端无文件） |
| 手动**导出**（Export）的 JSON | 自己选择的位置 |

该目录下还会有 `workflows\.index.json`，用于记录工作流的收藏/书签索引，不是工作流本身。

### 小结

| 主题 | 结论 |
|------|------|
| 图片输出 | `d:\ComfyUI\output\`，可用 `--output-directory` 改 |
| 耗时 | 冷启动首张约 39 分钟（含加载）；模型已缓存时更快，瓶颈是 CPU 算力与内存 |
| 内存 | 模型权重固定一份（约 31 GB），多张图不会翻倍；风险来自结果缓存与系统其他程序 |
| 参数 | batch 由 latent 决定（当前 1）；seed + `control_after_generate` 控制出图是否重复 |
| 配置 | 均在 `user\` 目录下的 JSON / SQLite / 工作流文件 |

---

## 2026-09-30 · Z-Image 出图失败：VAELoader 误用 pixel_space

### 现象

将分辨率降到 512×512 后重新提交，8 步采样全部完成（约 40 分钟），但任务最终失败，没有输出图片。

### 处理过程

**1. 从 API 读取失败详情**

终端的进度条只能看到采样过程，失败原因需要通过 ComfyUI 的历史记录接口获取：

```powershell
$r = Invoke-RestMethod "http://127.0.0.1:8188/history" -TimeoutSec 20
$r.PSObject.Properties | Select-Object -Last 1 | ForEach-Object { $_.Value.status }
```

异常栈的关键部分：

```text
File "D:\ComfyUI\execution.py", line 314, in process_inputs
    result = f(**inputs)
File "D:\ComfyUI\nodes.py", line 1707, in save_images
    img = Image.fromarray(np.clip(i, 0, 255).astype(np.uint8))
File "D:\ComfyUI\venv\Lib\site-packages\PIL\Image.py", line 3435, in fromarray
    raise TypeError(msg) from e
```

失败点在 `SaveImage` 节点 —— 说明采样与 VAE 解码都已执行完毕，只是写文件时报错。

**2. 检查工作流中的 VAE 配置**

```json
'63': {'inputs': {'vae_name': 'pixel_space'}, 'class_type': 'VAELoader'}
```

**3. 确认 pixel_space 是什么**

查看 [comfy/pixel_space_convert.py](comfy/pixel_space_convert.py)：

```python
class PixelspaceConversionVAE(torch.nn.Module):
    def decode(self, samples, *_args, **_kwargs):
        return samples
```

它是内置的"假 VAE"，**不做任何解码，原样返回输入**。Z-Image 的 latent 是 16 通道张量，直接交给 `Image.fromarray`（PIL 只接受最后一维为 1/3/4 的 HWC 数组）必然抛 TypeError。

**4. 确认缺少真正的 VAE 文件**

```
d:\ComfyUI\models\vae\
  - put_vae_here      ← 只有占位文件，没有 ae.safetensors
```

Z-Image Turbo 需要 Flux 的 VAE（`ae.safetensors`，约 335 MB）。本地没有该文件时，`VAELoader` 下拉框里只能选到 `pixel_space`。

**5. 补上 VAE 并验证**

```powershell
curl.exe -L -o "d:\ComfyUI\models\vae\ae.safetensors" `
  "https://hf-mirror.com/Comfy-Org/z_image_turbo/resolve/main/split_files/vae/ae.safetensors"
```

下载后校验大小（335,304,388 字节，与远程一致），重启服务，再查询节点定义确认选项已更新：

```powershell
(Invoke-RestMethod "http://127.0.0.1:8188/object_info/VAELoader").VAELoader.input.required.vae_name[0]
# 输出：ae.safetensors, pixel_space
```

### 结论

失败原因是 `VAELoader` 使用了内置的 `pixel_space` 假 VAE，解码输出仍是 16 通道 latent，`SaveImage` 无法把它转成图片。修复方式是下载 `ae.safetensors` 放入 `models\vae\`，并在工作流里改选它。

经验：

- Z-Image Turbo 依赖 Flux VAE（`ae.safetensors`），属于必需文件，与扩散模型、文本编码器同等重要；
- `pixel_space` 只适用于模型本身直接输出像素空间的场景，不适用于常规 latent 模型；
- 排查"采样正常但任务失败"这类问题时，ComfyUI 的 `/history` 接口比终端日志更直接，能拿到完整异常栈。

### 附：本次运行的其他实测数据

**FP8 在 CPU 上不省内存**

日志显示 fp8 权重在 CPU 模式下被转成 float32 再计算：

```text
[INFO] model weight dtype torch.float32, manual cast: None
[INFO] loaded completely;  23479.08 MB loaded
```

| 模型文件 | 磁盘大小 | 加载后内存 |
|----------|----------|------------|
| z_image_turbo_bf16.safetensors | 11.46 GB | 约 22.9 GB（转 fp32） |
| z_image_turbo_fp8_e4m3fn.safetensors | 5.73 GB | 约 23.5 GB（转 fp32） |

结论：CPU 模式下换 FP8 对内存没有收益，两者加载后都膨胀到约 23 GB。

**分辨率对耗时的影响**

| 分辨率 | 单步耗时 | 8 步总计 |
|--------|----------|----------|
| 1024×1024 | 991.86 秒（约 16.5 分钟） | 约 2 小时 12 分 |
| 512×512 | 298.61 秒（约 5 分钟） | 约 40 分钟 |

**本次工作流的有效配置**

| 节点 | 取值 |
|------|------|
| UNETLoader | `z_image_turbo_fp8_e4m3fn.safetensors`（`weight_dtype: default`） |
| CLIPLoader | `qwen_3_4b.safetensors`，`type: lumina2` |
| VAELoader | 应改为 `ae.safetensors`（原为 `pixel_space`） |
| EmptySD3LatentImage | 512 × 512，batch 1 |
| ModelSamplingAuraFlow | shift 3.0，sampling flow |
| KSampler | seed 42，steps 8，cfg 1.0，`res_multistep` / `simple`，denoise 1.0 |

---

## 2026-09-30 · 服务运行中崩溃的排查（Z-Image 加载时访问违规）

### 现象

在 Web UI 提交工作流后，http://localhost:8188 无法访问，浏览器提示连接失败。

### 处理过程

**1. 确认服务是否还活着**

```powershell
Get-NetTCPConnection -LocalPort 8188 | Select-Object LocalAddress,LocalPort,State,OwningProcess
Get-Process -Name python
```

结果是 8188 端口无任何监听记录，也没有残留的 python 进程 —— 服务进程是整体退出，而不是端口占用或防火墙问题。

**2. 查看服务终端输出**

进程退出码为 `3221225477`，即十六进制 `0xC0000005`（ACCESS_VIOLATION，内存访问违规），并带有崩溃调用栈：

```text
[INFO] got prompt
[INFO] Requested to load ZImageTEModel_
[INFO] loaded completely;  7672.25 MB loaded, full load: True
[INFO] Interrupting prompt 8ad16088-...
[INFO] Processing interrupted
[INFO] Prompt executed in 37.21 seconds
[INFO] got prompt

Windows fatal exception: access violation

Stack (most recent call first):
  File "...\torch\storage.py", line 480 in __getitem__
  File "D:\ComfyUI\comfy\utils.py", line 173 in load_torch_file
  File "D:\ComfyUI\comfy\sd.py", line 2426 in load_diffusion_model
  File "D:\ComfyUI\nodes.py", line 1013 in load_unet
  File "D:\ComfyUI\execution.py", line 547 in execute
```

**3. 检查内存实况**

```powershell
$os = Get-CimInstance Win32_OperatingSystem
$os.TotalVisibleMemorySize / 1KB    # 总内存
$os.FreePhysicalMemory / 1KB        # 空闲内存
Get-CimInstance Win32_PageFileUsage # 页面文件
```

| 指标 | 数值 |
|------|------|
| 系统总内存 | 31976 MB |
| 崩溃后空闲内存 | 15502 MB（说明崩溃前其他程序已占用约 16 GB） |
| 页面文件 | 分配 17 GB，当时使用 5.53 GB |
| 需同时驻留 | 文本编码器 7.67 GB + 扩散模型 11.46 GB |

### 结论

崩溃点落在**读取 safetensors 张量数据**这一步（`load_torch_file`）：第一次提交加载了 Qwen 文本编码器（7672 MB）后被中断，模型常驻未释放；第二次提交加载 11.46 GB 的 Z-Image 扩散模型时，可用内存与页面文件都不宽裕，safetensors 的 mmap 映射页在缺页加载时失败，Windows 直接以访问违规终止整个进程 —— 所以看不到 Python 层的 `MemoryError`，只有进程凭空消失。

判定为**内存压力**导致的崩溃，不是模型文件损坏（文件已在同日另一次校验中确认完整）。

规避措施：

1. 运行前关闭占内存的程序（浏览器、IDE 等），保证空闲内存 20 GB 以上；
2. 页面文件改为「由系统管理」或手动加大到 32 GB 以上，避免 mmap 缺页失败；
3. 不要在加载模型途中中断后立刻重新提交任务；
4. 若仍崩溃，启动时追加 `--use-split-cross-attention` 降低注意力计算的内存峰值（启动日志中也给出了该提示）；
5. 硬件条件受限时改用更小或量化版本的模型。

恢复方式：重新启动服务即可，模型缓存会重建。

```powershell
.\venv\Scripts\python.exe main.py --cpu
```

---

## 2026-09-30 · 模型下载失败（Chrome `.crdownload`）的恢复

### 现象

用 Chrome 下载 Z-Image 所需的两个模型时，浏览器提示「无法从网站提取文件」，随后提示「下载失败」。但下载目录中确实留下了两个体积很大的残留文件：

```
d:\Users\chengkai022\Downloads\未确认 208784.crdownload
d:\Users\chengkai022\Downloads\未确认 57033.crdownload
```

### 处理过程

**1. 弄清 `.crdownload` 是什么**

它是 Chrome 未完成下载时保留的临时文件，内容就是已接收的原始字节流。**Chrome 报「下载失败」并不等于数据不完整**，只要字节数够，直接改名即可当正常模型使用。

**2. 离线校验完整性**

safetensors 的文件结构自带长度信息，无需联网比对哈希：

```
[0:8]                 header 长度（小端 uint64）
[8:8+header_len]      JSON header，每个张量含 dtype / shape / data_offsets
[8+header_len:]       张量数据区
```

若 `8 + header_len + 各张量 data_offsets 最大值 == 文件实际大小`，即说明文件数据完整。校验脚本（用 venv 的 python 执行）：

```python
import json, os, struct

for p in [r"d:\Users\chengkai022\Downloads\未确认 57033.crdownload",
          r"d:\Users\chengkai022\Downloads\未确认 208784.crdownload"]:
    size = os.path.getsize(p)
    with open(p, "rb") as f:
        n = struct.unpack("<Q", f.read(8))[0]
        header = json.loads(f.read(n))
    header.pop("__metadata__", None)
    end = max(v["data_offsets"][1] for v in header.values())
    keys = list(header)
    print(os.path.basename(p), size, "complete:", size == 8 + n + end,
          "missing:", size - (8 + n + end))
    print("  tensors:", len(header), "first_keys:", keys[:4])
```

**3. 校验与识别结果**

| 原文件 | 大小 | 张量数 | 特征键名 | 判定为 | 完整性 |
|--------|------|--------|----------|--------|--------|
| 未确认 57033.crdownload | 11.46 GB | 453 | `cap_embedder` / `x_embedder` / `x_pad_token` | Z-Image 扩散模型（DiT） | 完整（缺失 0 字节） |
| 未确认 208784.crdownload | 7.49 GB | 398 | `model.embed_tokens` / `model.layers.N.self_attn` | Qwen3 文本编码器 | 完整（缺失 0 字节） |

识别依据是张量键名对应的结构：DiT 侧含 caption / latent embedder，文本编码器侧含 token embedding 与逐层 self-attention。

**4. 恢复文件（移动 + 重命名）**

```powershell
Move-Item -LiteralPath "d:\Users\chengkai022\Downloads\未确认 57033.crdownload" `
          -Destination "d:\ComfyUI\models\diffusion_models\z_image_turbo_bf16.safetensors"
Move-Item -LiteralPath "d:\Users\chengkai022\Downloads\未确认 208784.crdownload" `
          -Destination "d:\ComfyUI\models\text_encoders\qwen_3_4b.safetensors"
```

### 结论

两个文件的字节数都与 header 声明的总长度完全吻合，Chrome 只是没来得及完成改名，直接恢复即可正常使用。

几点注意：

- 源文件在项目工作目录之外，上述命令需在授权（沙箱外）模式下执行；
- 使用**移动**而非复制，同盘移动瞬间完成且不额外占用空间；
- 模型列表在服务启动时扫描，文件放入后需重启服务才能被识别。

---

## 2026-09-29 · 独立 Python 环境搭建、依赖安装与服务启动

### 目标

为 ComfyUI（`d:\ComfyUI`）创建独立的 Python 环境（python3），安装项目依赖并启动服务。

### 处理过程

**1. 检查系统 Python**

```powershell
python --version      # Python 3.12.1
python3 --version     # 命令不存在（Windows 无 python3 命令）
py -3 --version       # Python 3.12.1
```

项目 [pyproject.toml](pyproject.toml) 要求 `requires-python = ">=3.10"`，版本满足。

**2. 首次创建虚拟环境**

```powershell
python -m venv venv --upgrade-deps
```

创建成功，但此时并未察觉该环境是**基于 32 位 Python 3.12**，隐患在后面才暴露。

**3. 激活环境并安装依赖（首次失败）**

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

失败：

```
Looking in indexes: https://mirrors.aliyun.com/pypi/simple/
ERROR: Could not find a version that satisfies the requirement torch (from versions: none)
ERROR: No matching distribution found for torch
```

**4. 排查**

```powershell
python -m pip config list     # global.index-url=https://mirrors.aliyun.com/pypi/simple/
python -m pip --version       # 注意：pip 来自 32 位全局 Python 的 site-packages
.\venv\Scripts\python.exe -c "import sys,platform;print(platform.architecture()[0])"   # 32bit
py -0p                        # 列出系统所有解释器
```

排查结论：

1. 终端中**激活状态不跨命令保留**，`python` 实际解析到了全局 32 位解释器；
2. 更根本的问题：**已创建的 venv 本身就是 32 位**（`[MSC v.1937 32 bit (Intel)]`），而 torch 在 PyPI 上没有 32 位 wheel，无论用哪个源都装不上；
3. 系统可用解释器：

| 版本 | 路径 | 架构 |
|------|------|------|
| 3.12（默认，`py -3`） | `D:\Users\chengkai022\AppData\Local\Programs\Python\Python312-32\python.exe` | 32 位 |
| 3.11.16 | `D:\Users\chengkai022\AppData\Roaming\uv\python\cpython-3.11.16-windows-x86_64-none\python.exe` | **64 位** |

4. 同时确认 `nvidia-smi` 不存在，本机无 NVIDIA GPU。

经与用户确认后确定方案：**用现有 64 位 CPython 3.11.16 重建环境，安装 CPU 版 torch**。

**5. 重建虚拟环境**

```powershell
Remove-Item -Recurse -Force .\venv
& "D:\Users\chengkai022\AppData\Roaming\uv\python\cpython-3.11.16-windows-x86_64-none\python.exe" -m venv .\venv --upgrade-deps
```

验证结果：

```
version: 3.11.16 (main, Aug 14 2026, 15:34:35) [MSC v.1944 64 bit (AMD64)]
arch: 64bit
pip 26.2.1 from D:\ComfyUI\venv\Lib\site-packages\pip (python 3.11)
```

**6. 安装 CPU 版 torch**

```powershell
.\venv\Scripts\python.exe -m pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu --progress-bar off
```

结果：`torch-2.14.0+cpu`、`torchvision-0.29.0+cpu` 安装成功（torch wheel 约 124 MB）。

**7. 安装项目依赖（镜像源缺包，改用官方源）**

先按默认的阿里云镜像源安装，失败：

```
ERROR: Could not find a version that satisfies the requirement comfy-kitchen==0.2.36
       (from versions: 0.1.2 ... 0.2.35)
```

原因是阿里云镜像尚未同步 `comfy-kitchen 0.2.36`（最高仅 0.2.35），而 [requirements.txt](requirements.txt) 锁定了该版本。改用 PyPI 官方源重跑后成功：

```powershell
.\venv\Scripts\python.exe -m pip install -r requirements.txt --progress-bar off -i https://pypi.org/simple
```

已安装的 torch 被自动复用（`Requirement already satisfied: torch (2.14.0+cpu)`），未被覆盖。

**8. 验证**

依赖导入检查通过（`torch: 2.14.0+cpu`、`cuda available: False`、`transformers: 5.17.0`），随后启动检查：

```powershell
.\venv\Scripts\python.exe main.py --cpu --quick-test-for-ci
```

退出码 0，节点加载与数据库迁移（`0001_assets` → `0008_drop_asset_meta`）均正常完成。

**9. 启动服务**

```powershell
.\venv\Scripts\python.exe main.py --cpu
```

本次启动日志关键输出：

```text
[INFO] Total VRAM 31976 MB, total RAM 31976 MB
[INFO] pytorch version: 2.14.0+cpu
[INFO] Set vram state to: DISABLED
[INFO] Device: cpu
[INFO] Python version: 3.11.16 (main, Aug 14 2026, 15:34:35) [MSC v.1944 64 bit (AMD64)]
[INFO] ComfyUI version: 0.37.0
[INFO] [Prompt Server] web root: D:\ComfyUI\venv\Lib\site-packages\comfyui_frontend_package\static
[INFO] Using RAM pressure cache.
[INFO] Starting server
[INFO] To see the GUI go to: http://127.0.0.1:8188
```

启动日志中以下提示均属正常，无需处理：

| 日志 | 说明 |
|------|------|
| `No module named 'triton'` / `PyTorch ROCm/HIP runtime not available` | comfy-kitchen 的后端探测，CPU 环境下按预期不可用 |
| `No OpenGL_accelerate module loaded` | 可选加速模块缺失，仅影响性能 |
| `On windows we are currently forcing single GPU mode ...` | Windows 平台默认行为，与 CPU 模式无关 |

### 结论

过程中共遇到 5 个问题，对策如下：

| # | 问题 | 原因 | 解决方案 |
|---|------|------|----------|
| 1 | `python3` 命令不存在 | Windows 无 `python3` 命令 | 使用 `python`（或 `py -3`） |
| 2 | 找不到 torch | venv 基于 32 位 Python，torch 无 32 位 wheel | 改用 64 位 CPython 3.11.16 重建 venv |
| 3 | 依赖装不进目标环境 | 终端激活状态不跨命令保留，命令实际使用了全局解释器 | 统一使用 `.\venv\Scripts\python.exe` 绝对路径调用 |
| 4 | 找不到 `comfy-kitchen==0.2.36` | 阿里云镜像未同步该版本（最高 0.2.35） | 改用 `-i https://pypi.org/simple` 官方源 |
| 5 | 无 GPU 加速 | 本机无 NVIDIA GPU | 按用户确认安装 CPU 版 torch |

最终环境：

| 项目 | 结果 |
|------|------|
| 虚拟环境路径 | `d:\ComfyUI\venv` |
| 解释器 | CPython 3.11.16（64 位，uv 提供的独立发行版） |
| pip | 26.2.1 |
| torch | 2.14.0+cpu |
| torchvision | 0.29.0+cpu |
| 依赖安装 | `requirements.txt` 全部安装成功，共约 79 个包 |
| 服务地址 | http://127.0.0.1:8188（CPU 模式，默认端口 8188） |

其他关键包版本：transformers 5.17.0、tokenizers 0.23.2、numpy 2.4.6、Pillow 12.3.0、scipy 1.17.1、av 18.1.0、comfy-kitchen 0.2.36、comfy-aimdo 0.5.5、pydantic 2.13.5、SQLAlchemy 2.1.1、alembic 1.20.0。

日常使用：

```powershell
# 激活环境（仅对当前 PowerShell 窗口有效）
.\venv\Scripts\Activate.ps1

# CPU 模式启动（本机无 NVIDIA GPU，必须带 --cpu）
.\venv\Scripts\python.exe main.py --cpu

# 停止服务：在运行服务的终端按 Ctrl+C
```

后续注意：

1. 重装依赖时，直接执行 `pip install -r requirements.txt` 会因阿里云镜像缺 `comfy-kitchen==0.2.36` 而失败，请追加 `-i https://pypi.org/simple`，或单独先安装该包；
2. 重装依赖时注意**不要覆盖已安装的 CPU 版 torch**，如被覆盖需重新执行第 6 步的 CPU 索引命令；
3. [.gitignore](.gitignore) 中已有 `venv*/` 规则，虚拟环境不会被提交到仓库；
4. 项目另有一个可选的 [manager_requirements.txt](manager_requirements.txt)（ComfyUI Manager 依赖），本次未安装。
