# MTProxy ARM64 Native Port

This is a fork of [TelegramMessenger/MTProxy](https://github.com/TelegramMessenger/MTProxy) with patches to compile and run natively on **ARM64 (aarch64)** Linux.

## Changes Made

| File | Change |
|------|--------|
| `Makefile` | Removed x86-specific flags (`-mpclmul`, `-march=core2`, `-mfpmath=sse`, `-mssse3`) |
| `common/precise-time.h` | Added ARM64 `rdtsc()` via `mrs cntvct_el0` |
| `common/cpuid.c` | Added ARM64 stub returning zero feature flags (software fallback) |
| `common/crc32c.c` | Guarded x86 SSE4.2/CLMUL functions with `#ifdef __x86_64__` |
| `common/crc32.c` | Guarded all CLMUL functions with `#ifdef __x86_64__` |
| `common/server-functions.h` | Replaced `mfence` with `__sync_synchronize()` on ARM |
| `common/mp-queue.c` | Replaced `mfence` with `__sync_synchronize()` on ARM |
| `net/net-events.c` | Guarded `#include <sys/io.h>` with `#ifdef __x86_64__` |
| `common/pid.c` | Removed PID > 65535 assertion (modern Linux uses larger PIDs) |

## Build

```bash
apt install git curl build-essential libssl-dev zlib1g-dev
make
```

The binary will be at `objs/bin/mtproto-proxy`.

## Run

```bash
# Get proxy secret and config
curl -s https://core.telegram.org/getProxySecret -o proxy-secret
curl -s https://core.telegram.org/getProxyConfig -o proxy-multi.conf

# Generate a secret
head -c 16 /dev/urandom | xxd -ps

# Start the proxy
./objs/bin/mtproto-proxy -u nobody -p 8888 -H 443 -S <secret> --aes-pwd proxy-secret proxy-multi.conf -M 1
```
