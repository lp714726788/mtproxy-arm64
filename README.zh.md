# MTProxy ARM64 原生移植版

本项目是 [TelegramMessenger/MTProxy](https://github.com/TelegramMessenger/MTProxy) 的分支，进行了 ARM64（aarch64）Linux 原生编译移植。

官方 MTProxy 代码大量使用了 x86 专属指令集（SSE4.2、CLMUL、AES-NI、RDTSC 等），无法直接在 ARM64 架构上编译。本仓库通过条件编译和替代实现，使其能在 ARM64 上原生运行，**无需 QEMU 模拟**，性能更高、资源占用更低。

## 改动清单

| 文件 | 修改内容 |
|------|----------|
| `Makefile` | 移除 x86 专属编译参数（`-mpclmul`、`-march=core2`、`-mfpmath=sse`、`-mssse3`） |
| `common/precise-time.h` | 添加 ARM64 `rdtsc()` 实现（`mrs cntvct_el0` 读取周期计数器） |
| `common/cpuid.c` | 添加 ARM64 虚函数桩，CPU 特性标志全部返回 0，强制使用软件回退算法 |
| `common/crc32c.c` | 用 `#ifdef __x86_64__` 保护 x86 SSE4.2/CLMUL 专属函数 |
| `common/crc32.c` | 用 `#ifdef __x86_64__` 保护所有 CLMUL 加速函数 |
| `common/server-functions.h` | 将 `mfence` 指令替换为 ARM 兼容的 `__sync_synchronize()` |
| `common/mp-queue.c` | 同上，替换 `mfence` |
| `net/net-events.c` | 用 `#ifdef __x86_64__` 保护 x86 专属头文件 `<sys/io.h>` |
| `common/pid.c` | 移除 PID > 65535 的断言（现代 Linux PID 上限可达 4194304） |

## 编译

```bash
# 安装编译依赖
apt install git curl build-essential libssl-dev zlib1g-dev

# 编译
make

# 二进制文件在
ls -l objs/bin/mtproto-proxy
```

## 运行

```bash
# 获取代理密钥和配置文件
curl -s https://core.telegram.org/getProxySecret -o proxy-secret
curl -s https://core.telegram.org/getProxyConfig -o proxy-multi.conf

# 生成一个 16 字节的随机密钥
head -c 16 /dev/urandom | xxd -ps

# 启动代理（单 worker 模式）
./objs/bin/mtproto-proxy \
  -u nobody \
  -p 8888 \
  -H 443 \
  -M 1 \
  -C 60000 \
  --aes-pwd proxy-secret \
  proxy-multi.conf \
  --allow-skip-dh \
  -S <你的密钥>

# 如需 IPv6 支持，添加 -6 参数
```

## Systemd 服务示例

```ini
[Unit]
Description=MTProxy (ARM64)
After=network.target

[Service]
Type=simple
ExecStart=/opt/mtproto-proxy/mtproto-proxy \
  -p 2398 -H 2083 -M 1 -C 60000 \
  --aes-pwd /opt/mtproto-proxy/proxy-secret \
  -u nobody \
  /opt/mtproto-proxy/proxy-multi.conf \
  --allow-skip-dh \
  --nat-info "内网IP:外网IP" \
  -S <你的密钥>
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 性能说明

- 本项目使用软件回退算法替代 x86 硬件加速（CRC32、CLMUL），在 ARM64 上性能略低于 x86 原生版本，但避免了 QEMU 模拟的开销
- 实测 ARM64 Oracle Cloud 实例上，8MB 内存即可稳定运行
- 如需更高性能，可考虑使用 [seriyps/mtproto-proxy](https://github.com/seriyps/mtproto-proxy)（Erlang 实现，支持 ARM64 多平台 Docker 镜像）

## 链接格式

```
tg://proxy?server=你的服务器IP&port=端口&secret=16进制密钥
dd<密钥>  # 添加 dd 前缀启用 fake-TLS 模式，对抗 ISP 探测
```