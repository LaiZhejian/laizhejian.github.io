---
title: 'Flakes & chezmoi 配置使用'
date: 2025-11-10
permalink: /posts/2025/11/10/tools/
tags:
  - Notes
---

# Tool 使用

## Flakes

```shell
sh <(curl -L https://nixos.org/nix/install)
git clone --recursive https://github.com/LaiZhejian/flakes
cd flakes
git submodule update --init --recursive # 如果上一步没使用--recursive需补上
```

* For WSL or command-line Linux systems 

  ```shell
  sudo nix --extra-experimental-features 'nix-command flakes' run home-manager/master -- switch --flake ".?submodules=1#dream"
  ```

  In `/etc/nix/nix.conf`, add the below content

  ```shell
  experimental-features = nix-command flakes
  substituters = https://mirrors.ustc.edu.cn/nix-channels/store https://cache.nixos.org/
  ```

* For Mac

  ```shell
  sudo nix --extra-experimental-features 'nix-command flakes' run nix-darwin -- switch --flake ".?submodules=1#darwin" # setup
  sudo darwin-rebuild switch --flake ".?submodules=1#darwin" # rebuild
  ```

### Configuration

specific指代具体某个nix配置，platform指代宏观的如darwin（mac自己的preference）, desktop, command line 

* `flake.nix` 存储全局nix的配置，包括darwin，dream使用哪些具体的nix配置 （需要添加新specific改这里），组装【platform的nix】和【specific的nix】
* `configurations/{specific}/default.nix` 存储对应specific的nix配置（需要添加新specific改这里）
* `home/presets/{platform}` 存储对应platform的nix配置 （增添app）
* `modules/{platform}/{app}`修改platform下对应app的具体配置 （修改app）

## chezmoi

1. 初始化`chezmoi init https://github.com/LaiZhejian/dotfiles` 

   其中的文件`.chezmoi.toml.tmpl`允许使用明文加密和解析

   ```toml
   {{ $passphrase := promptStringOnce . "passphrase" "passphrase" -}}
   
   encryption = "gpg"
   [data]
       passphrase = {{ $passphrase | quote }}
   [gpg]
       symmetric = true
       args = ["--batch", "--passphrase", {{ $passphrase | quote }}, "--no-symkey-cache", "--quiet"]
   
   ```

2. `chezmoi apply`解密配置细腻下
3. `chezmoi add [--encrypt]`添加被管理文件，可选加密
