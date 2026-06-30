---
date: '2026-06-28'
draft: false
title: 'Debug：IntelliJ IDEA 在 NixOS 上无法编译 / 自动补全'
categories: ["NixOS"]
tags: ["nix", "idea", "nix-ld", "debug", "FHS"]
---

## 现象

在 NixOS 上用 IDEA 打开一个 Gradle/Java 项目后：

- 没有自动补全
- Build 失败
- IDEA 报错：

```
Abnormal build process termination:
Could not start dynamically linked executable: /home/feng/.jdks/azul-17.0.19/bin/java
NixOS cannot run dynamically linked executables intended for generic
linux environments out of the box.
```

## 排查

IDEA 试图启动自己下载的 Azul JDK 17（`~/.jdks/azul-17.0.19/bin/java`）来跑 Build Process。

手动试一下：

```bash
~/.jdks/azul-17.0.19/bin/java -version
# Could not start dynamically linked executable
```

确认了 — 这个二进制在 NixOS 上根本无法运行。

JDK 本身没问题，是 **NixOS 没有标准 Linux 的动态链接器**：

```bash
ls /lib/ld-linux-x86-64.so.2
# No such file or directory
ls /lib64/ld-linux-x86-64.so.2
# No such file or directory
```

Linux 上几乎所有原生程序（包括 Java 虚拟机）启动时都需要这个文件。通用 Linux 发行版（Ubuntu、Arch 等）都有，但 NixOS 把所有库放在 `/nix/store/<hash>-name/` 下，不存在传统的 `/lib` 或 `/usr/lib`。

通用二进制在编译时硬编码了 `/lib64/ld-linux-x86-64.so.2` 作为链接器路径。内核启动程序时先找链接器 → 找不到 → 拒绝启动。

## 解决

启用 `nix-ld` — NixOS 官方提供的机制，用于运行非 Nix 打包的通用 Linux 二进制。

```nix
# /etc/nixos/configuration.nix 或 flake 的 host 配置
programs.nix-ld.enable = true;
```

重建并注销重新登录：

```bash
sudo nixos-rebuild switch
```

清理 IDEA 之前下载的无效 JDK（可选，IDEA 会重新下载）：

```bash
rm -rf ~/.jdks/
```

重启 IDEA，Build Process 正常启动，自动补全和编译恢复。

## 原理

### 正常 Linux 程序启动流程

```
内核加载 ELF 二进制
  → 读 PT_INTERP 段 → "/lib64/ld-linux-x86-64.so.2"
    → 动态链接器接管
      → 扫 LD_LIBRARY_PATH, /lib, /usr/lib
        → 找到所有 .so
          → 链接完成，程序启动
```

### NixOS 下，nix-ld 做了什么

`nix-ld` 做了两件事：

1. **提供链接器 symlink**

   `/lib64/ld-linux-x86-64.so.2` → 指向 Nix store 里的 glibc 动态链接器

2. **设 `NIX_LD_LIBRARY_PATH`**

   指向 Nix 提供的一组常用 `.so` 库路径，供链接器检索。

### 为什么 nix-ld 默认不开启

它违背了 NixOS 的核心理念。

**Nix 打包方式**：每个程序编译时就 RPATH 硬编码所有依赖，精确到 `/nix/store/<hash>-name/`。
不存在运行时 "碰巧找到" 某个库 — 依赖被明确声明、版本锁定、可复现。

**nix-ld 方式**：给一套模糊的库路径让未经打包的二进制在运行时 "自求多福"：

- 库版本不精确：二进制编译时链 glibc 2.38，Nix 提供的是 2.40，ABI 兼容就没事，不兼容就随缘崩
- 无人声明依赖：缺库时手动排查、手动加
- 不可复现：库版本随系统更新而变

所以 nix-ld 默认关闭是 **故意的**，不是忘了。NixOS 希望你用 `nix-shell`、`buildFHSEnv`、`steam-run` 等原生方案，而非偷懒跑通用二进制。

但对 IDEA、VS Code 插件等真实需求，为每个第三方二进制重新打包不现实。nix-ld 是**务实的妥协**，社区知道这一点才把功能做进来了，只是默认不亮。
