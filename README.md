# CMP 170HX：250W → 300W 功耗上限解锁流程

仓库只记录功耗修改步骤，不提供整套板卡驱动归档。

验证环境：CMP 170HX / GA100，PCI ID `10de:2082`，VBIOS `92.00.66.00.02`，Ubuntu 24.04，内核 `6.8.0-138-generic`，NVIDIA 开源模块 `615.71.09`。

方法：修改总板功耗策略的最大值，通过补丁驱动将 VBIOS 覆盖到 GSP 内存，再设置运行功耗 300W。**没有刷写 EEPROM。** 默认值保持 250W。

## 1. 保存原始状态与 VBIOS

先确认没有 GPU 任务，保存 `nvidia-smi -q`、`/proc/driver/nvidia/params`、当前驱动模块和模块配置。

若 nvflash 要求卸载驱动，先停止持有 GPU 的服务，卸载 NVIDIA 模块，仅执行两次读取：

```bash
sudo /path/to/nvflash --save original.rom
sudo /path/to/nvflash --save original-second-read.rom
sha256sum original.rom original-second-read.rom
```

之后恢复原模块与服务。两次读取必须一致。本次源文件长度为 `1044480` 字节，SHA256：

```text
9fd77d45b06db99c3a84b822bf8ac161a3a8697c17dc7e19b212e1652a3743e9
```

下文偏移仅适用于这份哈希匹配的 ROM，其他镜像必须重新解析。

## 2. 定位总板功耗策略

- BIT `P + 0x2c` 指向功耗策略表，格式 `30/2c/43`：版本 `0x30`，表头 44 字节，每项 67 字节。
- 表头第 10 字节选择策略 2。策略 2 使用总板功耗通道 0，单位 mW。
- 最小／默认／最大值为 `100000 / 250000 / 250000`，与 NVML 的 `100 / 250 / 250W` 一致。
- 通道表格式 `20/06/16/14`：版本 `0x20`，表头 6 字节，每项 22 字节，20 项。

原编辑器未识别设备 `2082`，桌面 Ampere 通道解析又要求 32 项，因此不能直接套用。下面通过严格源哈希限制执行已验证的修改。

## 3. 只修改最大值并导出加载镜像

最大值字段在 `0x45e45`，重复镜像对应 `0xa5e45`；写入 little-endian uint32 `300000`，取代 `250000`。默认和最小值不改。修复两个数据镜像末尾的加法校验字节，最终只有 8 个字节实际变化。

把两次读取的文件放在当前目录，执行 Python 3：

```python
from pathlib import Path
import hashlib
import struct

sha = lambda b: hashlib.sha256(b).hexdigest()
src = Path("original.rom").read_bytes()
assert src == Path("original-second-read.rom").read_bytes()
assert len(src) == 1044480
assert sha(src) == "9fd77d45b06db99c3a84b822bf8ac161a3a8697c17dc7e19b212e1652a3743e9"
assert src[0x5e00:0x47600] == src[0x65e00:0xa7600]
out = bytearray(src)
for offset in (0x45e45, 0xa5e45):
    assert struct.unpack_from("<I", src, offset)[0] == 250000
    struct.pack_into("<I", out, offset, 300000)
for start, end in ((0xc600, 0x47600), (0x6c600, 0xa7600)):
    assert sum(src[start:end]) % 256 == 0
    out[end - 1] = (out[end - 1] - sum(out[start:end])) & 255
    assert sum(out[start:end]) % 256 == 0
changed = {i for i, (a, b) in enumerate(zip(src, out)) if a != b}
allowed = set(range(0x45e45, 0x45e49)) | set(range(0xa5e45, 0xa5e49)) | {0x475ff, 0xa75ff}
assert len(changed) == 8 and changed <= allowed
assert sha(out) == "2436a798e519f46f5705d95f495698e759e5ba7b9d9c4f8c90682642cef9902b"
payload = bytes(out[:393216])
assert sha(payload) == "c51f8b9a88529b987110a5448061eb28ab12baccbb64c4a1a74148f4c85f108e"
for name, data in (("300w.rom", out), ("300w-loader.bin", payload),
                   ("stock-loader.bin", src[:393216])):
    with Path(name).open("xb") as f:
        f.write(data)
    print(name, len(data), sha(data))
```

`300w.rom` 是完整归档镜像；`300w-loader.bin` 保留第一个 NVGI bank，长度 `393216`。加载器从 legacy 起点 `0x5e00` 开始，实际向 GSP 发送 `369152` 字节。不使用桌面 Ampere 的 relocation 算法。

**不要使用 nvflash 将修改 ROM 刷回 EEPROM。**

## 4. 构建 VBIOS override 驱动

使用 [NVIDIA Linux VBIOS Vix Loader](https://github.com/Panchovix/nvidia-linux-vbios-vix-loader)，本次基础提交为 `7bf011e3a6100fd6b470917256f73cc18bda0586`。

`MATCHED_SOURCE` 必须指向本机可工作的同版本开源驱动源码，保留其原有配置和功能。不能仅凭版本号假设任意源码能替代当前驱动。

```bash
git clone https://github.com/Panchovix/nvidia-linux-vbios-vix-loader.git loader
git -C loader checkout 7bf011e3a6100fd6b470917256f73cc18bda0586
MATCHED_SOURCE=/path/to/working/driver-source-615.71.09
python3 loader/nvbios driver prepare \
  --source "$MATCHED_SOURCE" --destination "$PWD/driver-override" > prepare.json
make -C driver-override -j8 modules \
  SYSSRC=/lib/modules/6.8.0-138-generic/build > build.log 2>&1
```

产物：`driver-override/kernel-open/nvidia.ko`。加载器底层补丁原本已有 `IsGA100` 分支，经 `vbiosOverrideArgs` 传入覆盖。GA100 使用 `gsp_tu10x.bin`。

本流程直接加载已验证的 payload，不依赖原加载器高层设备识别自动接受 `2082`。主机端允许暂存也不代表固件一定接受，必须执行下一步实机验证。

## 5. 原始镜像对照与 300W 验证

备份可用模块并停止 GPU 任务，保留原模块参数，在补丁模块参数中额外加入以下功耗覆盖项：

```text
NVreg_RegistryDwordsPerDevice=pci=0000:01:00.0;RmLinuxVbiosOverride=2;RmLinuxVbiosOverrideSize=393216
NVreg_RegistryBinaryFilePerDevice=pci=0000:01:00.0;RmLinuxVbiosOverrideImage=615.71.09/vbios-override/test.bin
```

镜像位置：`/lib/firmware/nvidia/615.71.09/vbios-override/test.bin`。

先放 `stock-loader.bin`，卸载原模块、加载补丁模块，确认初始化正常且默认／最大为 `250/250W`；再卸载模块，换成 `300w-loader.bin` 并重新加载，确认变为 `250/300W`。不要覆盖系统原来的可用模块。

以上参数必须叠加到原环境参数，不能替代原配置；`insmod` 不会自动加载依赖，应按 `modinfo -F depends`、`modinfo -F softdep` 准备依赖。

```bash
nvidia-smi -q -d POWER
sudo dmesg | grep vbios_override
sudo nvidia-smi -pl 300
nvidia-smi --query-gpu=power.limit,power.default_limit,power.min_limit,power.max_limit --format=csv
```

期望：当前 `300W`、默认 `250W`、最小 `100W`、最大 `300W`。

日志 `369152 bytes from offset 0x5e00 staged for GSP` 仅证明主机暂存；最大值变为 300W、设置 300W 成功才是本次固件接受修改的实机证据。

## 6. 短测和五分钟烤机

先做计算正确性校验，再运行：

```bash
./gpu_burn -tc 300
```

另一个终端采样：

```bash
nvidia-smi --query-gpu=timestamp,power.draw,power.limit,temperature.gpu,clocks.sm,utilization.gpu \
  --format=csv -l 1 > power.csv
```

本次 GPU Burn 退出码 0、`GPU 0: OK`，计算和 ECC 错误为 0；高负载逐秒采样平均约 `234.97W`、瞬时峰值 `326.26W`、最高温度 `53°C`。300W 是请求和允许上限，不是恒定耗电目标，瞬时读数可越过它；本次短测不代表长期稳定性认证。

## 7. 开机加载与回退

实机验证后再建立仅负责功耗覆盖的开机服务：检查设备、内核、驱动及 payload 哈希，加载补丁模块和修改镜像，确认最大值 300W，最后执行 `nvidia-smi -pl 300`。

这是磁盘配置和驱动加载流程的持久化，**不是把 300W 硬编码进模块，也不是 EEPROM 固化**。已实测手动启用、回退和再次启用；未进行整机重启验证。

回退：停止 GPU 任务，禁用功耗覆盖启动项，停止持有 GPU 的服务，卸载补丁模块，再不带覆盖参数加载原来的可用模块并恢复服务。核对当前和最大功耗恢复 `250W`。

只停止 systemd unit 不会自动撤销已经加载到内存的覆盖。更换内核、驱动、板卡或 ROM 后需重新分析与验证。
