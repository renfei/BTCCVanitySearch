# VanitySearch（BTCC 版）

基于 [JeanLucPons/VanitySearch](https://github.com/JeanLucPons/VanitySearch) 修改，新增对 **BTCC（Bitcoin Classic）** 网络的支持，同时保持原有 BTC 功能不变。

通过 `-coin BTC|BTCC` 参数切换网络，默认为 BTC。

# 功能特性

- 支持 BTC 与 BTCC 双网络切换
- 支持 P2PKH、P2SH、BECH32 地址格式
- 支持分割密钥（split-key）靓号地址生成
- Fixed size arithmetic
- Fast Modular Inversion (Delayed Right Shift 62 bits)
- SecpK1 Fast modular multiplication (2 steps folding 512bits to 256bits using 64 bits digits)
- Use some properties of elliptic curve to generate more keys
- SSE Secure Hash Algorithm SHA256 and RIPEMD160 (CPU)
- Multi-GPU support
- CUDA optimisation via inline PTX assembly
- Seed protected by pbkdf2_hmac_sha512 (BIP38)

# BTCC 网络参数

| 参数 | 值 |
|------|----|
| P2PKH version byte | `0x1C`（28） |
| P2SH version byte  | `0x28`（40） |
| WIF version byte   | `0xBC`（188） |
| Bech32 HRP         | `cc` |
| P2PKH 地址首字符    | `C` |
| P2SH 地址首字符     | `H` |
| Bech32 地址前缀     | `cc1q` |

> **注意**：BTCC P2PKH 地址的**第二个字符**受版本字节约束，仅覆盖 Base58 索引 15–38，即字符范围 `G`–`f`（含大写 `G`–`Z` 和小写 `a`–`f`）。例如 `CT`、`CU`、`Ca`、`Cb` 是有效前缀，而 `Cr`、`Cs` 等不在该地址空间内，搜索不会有结果。

# 用法

```
VanitySearch [-check] [-v] [-u] [-b] [-c] [-gpu] [-stop] [-i inputfile]
             [-gpuId gpuId1[,gpuId2,...]] [-g g1x,g1y,[,g2x,g2y,...]]
             [-o outputfile] [-m maxFound] [-ps seed] [-s seed] [-t nbThread]
             [-nosse] [-r rekey] [-check] [-kp] [-sp startPubKey]
             [-rp privkey partialkeyfile] [-coin BTC|BTCC] [prefix]

 prefix: 要搜索的前缀（可包含通配符 '?' 或 '*'）
 -coin: 指定网络，BTC 或 BTCC（不区分大小写），默认 BTC
 -v: 打印版本
 -u: 搜索未压缩地址
 -b: 同时搜索压缩和未压缩地址
 -c: 前缀大小写不敏感搜索
 -gpu: 启用 GPU 计算
 -stop: 找到所有前缀后停止
 -i inputfile: 从文件读取前缀列表
 -o outputfile: 将结果输出到文件
 -gpu gpuId1,gpuId2,...: 指定使用的 GPU，默认为 0
 -g g1x,g1y,g2x,g2y,...: 指定 GPU kernel gridsize
 -m: 每次 kernel 调用最多找到的前缀数
 -s seed: 指定基础密钥的种子，默认随机
 -ps seed: 将种子与加密安全随机种子拼接后使用
 -t threadNumber: 指定 CPU 线程数，默认为 CPU 核心数
 -nosse: 禁用 SSE 哈希函数
 -l: 列出支持 CUDA 的设备
 -check: 对比 CPU 与 GPU kernel 的计算结果
 -cp privKey: 从私钥（十六进制）计算公钥
 -kp: 生成密钥对
 -rp privkey partialkeyfile: 从分割密钥信息重建最终私钥
 -sp startPubKey: 从指定公钥开始搜索（用于密钥分割）
 -r rekey: 重新生成密钥的间隔（MegaKey），默认禁用
```

# 使用示例

## 搜索 BTCC P2PKH 靓号地址

```sh
# 搜索以 "CT" 开头的 BTCC P2PKH 地址
$ ./VanitySearch -coin BTCC -stop CT
VanitySearch v1.20 [BTCC]
Difficulty: 2809
Search: CT [Compressed]
...
PubAddress: CT11Q86RqbMZBWohDabUfBXmimp8sPGFbn
Priv (WIF): p2pkh:UB3...（BTCC WIF，以 'U' 开头）
Priv (HEX): 0x...
```

## 搜索 BTCC Bech32（SegWit）靓号地址

```sh
# 搜索以 "cc1q00" 开头的 Bech32 地址
$ ./VanitySearch -coin BTCC -stop cc1q00
VanitySearch v1.20 [BTCC]
Difficulty: 1073741824
Search: cc1q00 [Compressed]
...
PubAddress: cc1q00qdt78gyjpzwdyheu323j0k3taed3xjjufng7
Priv (WIF): p2wpkh:U...
Priv (HEX): 0x...
```

## 搜索 BTC 地址（默认行为不变）

```sh
$ ./VanitySearch -stop 1abc
VanitySearch v1.20 [BTC]
...
PubAddress: 1abcv611Gy6Wx7eHncfZKAQUi56sYK2TX
Priv (WIF): p2pkh:K...
Priv (HEX): 0x...
```

## 生成 BTCC 密钥对

```sh
$ ./VanitySearch -coin BTCC -kp
Priv : U...（BTCC WIF 私钥）
Pub  : 03...（压缩公钥）
```

# 使用分割密钥为第三方生成靓号地址

适用场景：Alice 想要一个靓号地址但算力不足，Bob 有算力但不应知道 Alice 的私钥。

## 第一步：Alice 生成密钥对

```sh
$ ./VanitySearch -coin BTCC -s "AliceSeed" -kp
Priv : U...
Pub  : 03FC71AE1E88F143E8B05326FC9A83F4DAB93EA88FFEACD37465ED843FCC75AA81
```

Alice 保留私钥，将公钥和想要的前缀发送给 Bob。

## 第二步：Bob 使用 Alice 的公钥搜索

```sh
$ ./VanitySearch -coin BTCC -sp 03FC71AE1E88F143E8B05326FC9A83F4DAB93EA88FFEACD37465ED843FCC75AA81 -stop -o keyinfo.txt CT
```

生成的 `keyinfo.txt` 包含部分私钥，Bob 将文件发回 Alice。

## 第三步：Alice 重建完整私钥

```sh
$ ./VanitySearch -coin BTCC -rp U...（Alice 的私钥） keyinfo.txt

Pub Addr: CT...
Priv (WIF): p2pkh:U...
Priv (HEX): 0x...
```

# 编译

本项目使用 x86 内联汇编与 SSE 指令集，**不能在 ARM 架构下原生编译**。在 Apple Silicon Mac 上需借助 Rosetta 2 以 x86_64 模式运行。

## macOS x86_64（Intel Mac）

```sh
make
```

## macOS ARM（Apple Silicon，M 系列芯片）

需要先安装 Rosetta 2，再以 x86_64 模式编译和运行：

```sh
# 安装 Rosetta 2（如尚未安装）
softwareupdate --install-rosetta

# 以 x86_64 模式编译
arch -x86_64 make

# 运行时同样需要加 arch -x86_64 前缀
arch -x86_64 ./VanitySearch -coin BTCC -t 2 cc1q00000
```

> 每次执行都需要带 `arch -x86_64` 前缀，否则会因指令集不兼容而报错。

## Linux x86_64

```sh
# CPU-only 编译
make

# GPU 编译（根据实际硬件设置 CCAP）
make gpu=1 CCAP=8.6
```

## Windows x86_64

打开 `VanitySearch.sln`（Visual C++ 2017），在 Build -> Configuration Manager 中选择 Release 配置，编译即可。

# License

VanitySearch is licensed under GPLv3.
