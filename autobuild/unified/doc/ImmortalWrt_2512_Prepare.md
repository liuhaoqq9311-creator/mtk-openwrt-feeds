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
