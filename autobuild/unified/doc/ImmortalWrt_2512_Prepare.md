# ImmortalWrt 25.12 的 Filogic mac80211 prepare 适配

本地适配基线：

- ImmortalWrt `89dfac46c9f1eaeb56c6cf49eb289b4475f446f7`（`openwrt-25.12`，本地工作分支 `dev_wt9103`）。
- MediaTek feed `c16148f6e7f29ae08e5935215620f189fc56104e`（`main`）。
- 只准备 `filogic-mac80211`，不执行 build、release 或设备烧录。

## 修复范围

`25.12/patches-base/`：

- `1132`、`1133`、`1146`：匹配并保留 ImmortalWrt 的 `automount`，保持 MTK 的 overlay、PHY 和镜像改动。
- `3000`：匹配新的 fstools 依赖关系，保留 block-mount 的 `+fstools` 并添加 `+libfstools-bootparam`。
- `3600`：适配新增设备的 case 上下文，保留其他设备，只把 MTK RFB 移到其升级处理分支。
- `3704`：保持 MTK 去除默认 safexcel 包的意图，同时保留 ImmortalWrt 的 `wpad-openssl` 和 `bridger`。

`autobuild/unified/filogic/mac80211/25.12/patches-base/`：

- `0001`：删除重复添加 MT7915 WED 参数的 hunk。ImmortalWrt 已对 Filogic/MT7622 设置 `wed_enable=Y`，保留其平台条件；其他 MT76 改动仍应用。
- 删除 `0008`：ImmortalWrt 已默认选择完整 `wpad-openssl`，该补丁只负责从上游的 `wpad-basic-mbedtls` 切换，目标已满足。

这些上下文适配针对以上 ImmortalWrt 基线，未验证对原版 OpenWrt 的可应用性。

## 执行与恢复

在 `openwrt/` 目录执行：

```sh
bash ../mtk-openwrt-feeds/autobuild/unified/autobuild.sh filogic-mac80211 prepare
```

主机需要 `swig`。如果 ccache 默认目录不可写，可只为当前命令设置可写的 `CCACHE_DIR`。执行环境必须能够访问 feeds 远端；无需改全局代理配置。

prepare 包括 feeds 更新/安装、补丁应用、配置生成以及 PHY 固件下载/解包。配置工具和 autobuild 主机工具属于该阶段；完整工具链、内核和固件镜像构建不在本次验证范围。

失败不会回滚已经应用的补丁，也不会记录逐补丁续跑进度。不要对半完成状态直接重跑。框架 `clean` 会恢复主树及 feeds；其中 `mtk_openwrt_feed` 为指向本地 MTK 仓库的 src-link，因此未提交的 MTK 修复也会被它清理，应先保留这些修复。

## 验证边界

基础补丁和 WiFi Makefile 补丁已在从 ImmortalWrt HEAD 提取的临时副本中按序应用检查。完整 prepare 的最终结果以命令退出码和 `.ab/branch_name` 为准。prepare 成功不代表内核补丁在源码展开时全部适用，也不代表镜像构建或硬件功能通过。

### 2026-10-07 本地执行结果

- 完整 prepare 退出码为 0；`.ab/branch_name` 内容为 `filogic-mac80211`，日志只出现 prepare 阶段。
- 六个 feeds 已准备；MTK feed 为指向本地仓库的 src-link。
- `.config` 及 `.ab/filogic-mac80211_defconfig` 已生成，PHY 固件已经复制。
- 使用主机 SWIG 4.3.0，ccache 写入 `/tmp/router-mtk-ccache`；未修改全局代理。
- 复用其他本地树中的 linux-firmware 20260221 归档，SHA-256 与包声明一致：`bd19acc4c1a02548e09d3df67f987fe6e378df735bab138c1d9e917962056d94`。
- 修复补丁的 diff 检查通过（补丁上下文的空格加制表符不作为缩进错误）。OpenWrt 全树 diff 检查仍报告 MTK 原始 `wifi-scripts/files-ucode/usr/share/ucode/iwinfo.uc:118` 的一处尾随空白，未作为无关改动修复。
- 日志存于父工作区 `artifacts/mtk-prepare-2026-10-07/prepare.log.gz`。
- 未添加测试；没有进入完整 build/release，没有编译内核或生成/刷写固件镜像，硬件运行未验证。

### 2026-10-08 内核源码 prepare 修复

用户实际构建在 `999-crypto-04-extend-xfrm-packet-mode.patch` 失败：Linux 6.12.108 的 `xfrm_dev_direct_output()` 在 `local_out` 错误分支增加了 `kfree_skb(skb)` 及花括号，原删除 hunk 未包含这两行。同步调整 feed 来源及已导入目标树中的删除 hunk 和后续新文件行号，保留 MTK 删除整个 helper 的原语义，其他卸载逻辑不变。

原补丁在实际失败点源码上复现失败；修复补丁全部 hunk 以 `fuzz=0` 应用成功。目标树执行 `make target/linux/prepare V=s` 退出码 0，完整内核补丁序列通过，生成 prepared 标记且没有 `.rej`。当前 `.config` 逐字节保持；未自动重启完整固件构建或刷机。本次验证证明内核源码展开/补丁应用通过，不代表内核编译、固件或 IPsec 硬件功能已验证。日志与备份位于父工作区 `artifacts/wt9103-kernel-prepare-2026-10-08/`。

### 2026-10-08 fstools extroot 补丁兼容修复

MTK 的 `0001-add-support-for-dual-boot.patch` 先给 `block` 添加 `fstools-bootparam`，ImmortalWrt 既有 `100-fstools-support-extroot-for-non-MTD-rootfs_data.patch` 随后仍匹配旧链接行，导致 fstools prepare 的 CMakeLists.txt hunk 失败。目标树只将 bootparam 加入该 hunk 的删除/新增行，保留 extroot 添加 fstools 的原行为和双系统库。

新增 `25.12/patches-base/3001-fstools-align-extroot-with-dual-boot.patch`，在 3000 后适配既有 extroot 补丁，避免重新 prepare 导入旧上下文。此补丁针对已记录的 ImmortalWrt 基线，不声明兼容缺少该 extroot 补丁的原版 OpenWrt。已在临时副本以 fuzz=0 应用并确认结果与目标树逐字节相同；完整 fstools prepare 退出码 0，原 CMake hunk 不再失败。编译结果与验收边界见父工作区 `specs/004-wt9103-mac80211/validation.md`；日志与配置备份位于 `artifacts/wt9103-fstools-fix-2026-10-08/`。

### 2026-10-08 mt76 vendor 头文件补丁适配

实际构建在 `0125-cp-mtk-mt76-unify-vendor-commands-for-different-chip-ge.patch` 的 `mt7996/vendor.h` → `vendor.h` 首 hunk 失败：0053 已添加 SPDX 和 2025 版权头，0125 仍匹配无文件头的旧内容。同步目标树及 `autobuild/unified/filogic/mac80211/25.12/files/package/kernel/mt76/patches/` 来源：首 hunk 匹配现有头、保留原许可证与 2025 版权，增加原补丁的 2026 版权并重命名保护宏；更新同一文件后续 hunk 行号，vendor 命令逻辑不变。

完整 mt76 prepare 通过；重命名头文件七段在临时源码上修复前失败、修复后 fuzz=0 通过。执行 `make -k target/linux/prepare package/prepare V=s` 退出码 0，内核及当前配置的 191 个包 prepare 条目均通过（包含既有准备标记的复用）。该命令用于内核及 y/m 包的源码准备，不覆盖未选包和编译/运行错误；同包仍停在首个失败补丁，修复后应重跑。普通 `make prepare` 进入目标内核编译依赖链，不是批量 package prepare。

备份、批量日志与头文件前后对照存于父工作区 `artifacts/wt9103-mt76-prepare-2026-10-08/`；后续编译结果见 `specs/004-wt9103-mac80211/validation.md`。未启动 world 或刷机。
