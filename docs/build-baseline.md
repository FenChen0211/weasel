# 原版基线、GitHub Actions 与用户数据保护

## 本阶段状态

核对日期：2026-10-14（Asia/Shanghai）。用户已将构建策略调整为优先云端：本机不安装 Visual Studio、Build Tools、Boost、CMake 等编译依赖。

- 个人 fork：[`FenChen0211/weasel`](https://github.com/FenChen0211/weasel)。
- 原版源码：`rime/weasel@4aea1978a351e2ce41af20c8fb88a01ce9802de8`；获取时通过 `git ls-remote` 核验 master。
- 本地开发分支：`codex/ui-foundation`。`origin` 指向个人 fork，`upstream` 指向 `rime/weasel`；不向 upstream 推送。
- 本机仅拉取和阅读源码、创建文档和工作流。没有执行本机构建、安装输入法、注册 TSF DLL、重新部署或替换已有二进制。
- 新工作流：[`ui-baseline.yml`](../.github/workflows/ui-baseline.yml)，在云端独立 checkout 原版源码。
- 首次个人 fork 运行：[Actions run 38558777570](https://github.com/FenChen0211/weasel/actions/runs/38558777570)，工作流提交 `a9761117ecd0ee00b14d377ad50e9afd8431b40a`。**已完成，conclusion=success**；2026-10-14 19:45:05–19:53:07（Asia/Shanghai），空 Boost 缓存首次构建约 8 分钟。
- x64：57 warnings / 0 errors；Win32：31 warnings / 0 errors。两份 MSBuild 日志均为 Build succeeded，产物完整性、tracked source 与子模块前后版本检查全部通过。保留原版警告，没有为消警告改源码。
- [二进制、数据和符号](https://github.com/FenChen0211/weasel/actions/runs/38558777570/artifacts/11890728460)：artifact `11890728460`，50,450,719 字节，SHA256 `c5ae48ef8bc07a11b074ebb24761ae7ad5b69eb9109c3abc91600905f90a6c10`，保留至 2026-10-28。
- [构建证据](https://github.com/FenChen0211/weasel/actions/runs/38558777570/artifacts/11890938217)：artifact `11890938217`，396,219 字节，SHA256 `90dd010975968efaa3c830e7f0b14fec1dacd605dad8164a46f6279b7a8700d8`，保留至 2026-11-13。已下载校验并保存在本地忽略目录 `.cache/actions/38558777570/`，避免远端过期后丢失本次证据。
- 已读取归档中的 source-sha、build-environment、两份 msbuild 日志及 artifacts-sha256；确认编译源码是原版 `4aea1978...`，VS2022/v143 `14.44.35207`、SDK `10.0.19041.0`、Boost `1.84.0`，子模块前后快照一致。

UI 架构和后续改造边界见 [ui-architecture.md](ui-architecture.md)。

## 以实际工程和日志核实的构建方式

`INSTALL.md` 仍推荐 VS2017 和 XP support，不能作为当前环境选型的依据。实际源码、`.vcxproj`、`build.bat`、`env.vs2022.bat` 和 `.github/workflows/ci.yml` 表明当前路径如下：

| 项目 | 核实结果 |
| --- | --- |
| 云端 runner | upstream CI 使用 `windows-2022`，不是 `windows-latest` |
| 工具链 | VS2022，`BJAM_TOOLSET=msvc-14.3`，`PLATFORM_TOOLSET=v143` |
| SDK | upstream CI 显式指定 `10.0.19041.0` |
| 工程 | 根目录 `weasel.sln` + 各 `.vcxproj`；另有 xmake 路径，本阶段优先 MSBuild |
| C++ / runtime | WeaselUI C++17、Unicode，Release 静态 CRT `/MT` |
| Boost | CI 使用 1.84.0；env.vs2022 的缺省目录仍是 1.78.0，需要由 BOOST_ROOT 显式覆盖 |
| Windows UI 头文件 | 使用 ATL / WTL；WTL 头已放在 `include/wtl`。云端工作流额外核验 ATL/MFC 组件头存在 |
| 核心依赖 | upstream CI 调用 `get-rime.ps1 -use dev` 获取预编译 librime；不等于从 gitlink 版本编译 librime |
| 直接绘制依赖 | 使用方链接 d2d1、dwrite、d3d11、dxgi、dcomp、shcore 等 Windows SDK 库 |
| 通常构建顺序 | env.bat → Boost → librime headers/libs/DLL + OpenCC data → bundled data → render.js 生成 weasel.props → MSBuild |
| 默认平台顺序 | `build.bat weasel` 先 Release x64，再 Release Win32；`arm64` 额外启用 ARM、ARM64 和 ARM64X wrapper |
| 安装包 | `installer` 选项调用 NSIS。`build.bat all` 还编译引擎依赖并打包，不适合这次的最小基线 |

原版 `build.bat` 含退出已存在 WeaselServer 的行为；`get-rime.ps1 -use dev` 会查找并终止 WeaselServer，`-use weasel` 则会按注册表替换已安装的 Rime DLL。**本机没有运行这些脚本。**

### 上游成功记录（与个人 fork 新运行分开）

已读取 exact SHA 对应的 [upstream CI run 38225765635](https://github.com/rime/weasel/actions/runs/38225765635) 和 [MSBuild job 114753297381](https://github.com/rime/weasel/actions/runs/38225765635/job/114753297381) 的实际日志。该运行在 2026-10-12 完成：

- CI、MSBuild 和 xmake 的 conclusion 均为 success。
- MSBuild `17.14.60+43b635718`；编译器目录显示 MSVC `14.44.35207`，属于 VS2022/v143。
- runner image `windows-2022`，image version `20261004.326.1`；系统为 Windows Server 2022。
- Boost 1.84.0；Windows SDK 10.0.19041.0。
- librime 下载名为 `rime[-deps]-33e7814-Windows-msvc-x64/x86.7z`。
- ARM、ARM64、x64、Win32 的 MSBuild 均报告 Build succeeded / 0 Error(s)，最后产生 `weasel-0.17.4.92.4aea197-installer.exe` 并上传。

这是 upstream 已完成构建的证据；本次个人 fork 的构建结果由上面的新 run 单独记录。不能仅凭旧 CI 成功就声称新工作流已经通过。

## 递归子模块快照

执行了 `git submodule update --init --recursive`，结束后 `git submodule status --recursive` 各行均无 `-`、`+` 或 `U` 前缀；`git submodule foreach --recursive 'git status --porcelain'` 无修改。

| 路径 | 固定提交 |
| --- | --- |
| librime | `1c23358157934bd6e6d6981f0c0164f05393b497` |
| librime/deps/glog | `7b134a5c82c0c0b5698bb6bf7a835b230c5638e4` |
| librime/deps/googletest | `f8d7d77c06936315286eb55f8de22cd23c188571` |
| librime/deps/leveldb | `99b3c03b3284f5886f9ef9a4ef703d57373e61be` |
| librime/deps/leveldb/third_party/benchmark | `bf585a2789e30585b4e3ce6baf11ef2750b54677` |
| librime/deps/leveldb/third_party/googletest | `c27acebba3b3c7d94209e0467b0a801db4af73ed` |
| librime/deps/marisa-trie | `0d4e8ab58eec355facf8f65ff11ef811b330e373` |
| librime/deps/opencc | `e5d6c5f1b78e28a5797e7ad3ede3513314e544b7` |
| librime/deps/yaml-cpp | `f7320141120f720aecc4c32be25586e7da9eb978` |
| plum | `cab9ed35289f2022b6888503750d2dfb1851dbe0` |

## 个人 fork 云端流程

进入个人 fork 的 Actions → **UI fork Windows baseline** → **Run workflow**。

- `target=baseline`：源码固定为上面的 upstream SHA。工作流定义与编译源码分离，始终验证这一份原版。
- `target=fork`：编译手动触发时所选分支对应的 `github.sha`，供后续 UI 修改使用。
- 只手动触发，不自动发布 Release、安装、写入用户数据或运行输入法。工作流需存在于默认分支才能在页面发现。
- token 权限仅 `contents: read`，checkout 不持久化凭据，第三方 actions 固定到已核实的提交 SHA。

构建步骤：

1. 独立 `source/` checkout，`fetch-depth: 0`、`submodules: recursive`，检查全部 gitlink 和记录版本。
2. 激活 VS2022/v143 + SDK 10.0.19041.0，再复制仓库 `env.vs2022.bat` 为本次 env.bat。
3. 缓存 Boost 源码及 x64/Win32 静态库；key 包含编译器版本、SDK、平台和 build.bat 哈希。缓存未命中时从 Boost 官方下载并验证 SHA256，然后调用原版 `build.bat boost`。与 upstream 步骤相比，提前激活 MSVC，确保空缓存也能编译。
4. 下载固定的 librime 1.17.0 发布包（binary commit `33e7814`），验证四个 SHA256，复制构建所需文件。这里不调用带进程副作用的 get-rime 脚本。librime 预编译版本与源码 gitlink 版本的差别是刻意沿用 upstream CI 的依赖选择，不是改动子模块。
5. 调用原版 `build.bat data`，plum 的目标目录固定在 runner 的 `output/data`；记录生成的数据 SHA256。
6. 调用原版 `build.bat weasel`，编译 Release x64 / Win32。
7. 检查 TSF DLL、Server、Deployer、Setup 和两份 WeaselUI.lib；核验非 output 区域的 tracked source 未变及子模块提交未变；tracked diff 非空时保存补丁。此次 diff 为空，PowerShell 管道没有生成空补丁文件。
8. 上传 output 中的二进制、数据和符号（14 天），另存工具链、source SHA、submodule SHA、依赖及产物 SHA256、构建日志、生成的 weasel.props（30 天）。失败也上传已有 evidence。

该流程复用仓库原有 WinSparkle 文件（x64 为 0.8.1、Win32 为 0.6.0），不重编译 upstream CI 的补丁版 WinSparkle 0.9.2；因此这是原版 Weasel 源码的构建基线，不声称与官方安装包逐字节一致。此阶段不产生 NSIS 安装包，不覆盖 ARM / ARM64 / ARM64X，不执行 GUI 或输入功能测试。现有上游 Release 解决方案未启用两个 test 项目的 Release Build.0；单纯 solution build 也不等于运行测试。

### 依赖锁定与可复现范围

Boost 1.84.0 压缩包 SHA256：`81a4d10075731966477276c47324ecd9cac02da49899d36c48eba66e40014f25`，来自 [Boost 官方元数据](https://archives.boost.io/release/1.84.0/source/boost_1_84_0.7z.json)。

librime 包和哈希已写在工作流中，依据 [librime 1.17.0 Release](https://github.com/rime/librime/releases/tag/1.17.0) 的 asset digest，并和已通过的 upstream job 下载名对照。

源码、子模块、Boost、librime 二进制和 actions 版本固定；GitHub hosted runner image 会更新，plum 的 preset 数据仓库也可能更新，所以目前保证的是**可检查、可重复执行的构建流程**，不是位级可重现。每次记录 image / compiler 版本和数据哈希，后续若有严格要求再固定词典仓库提交及 runner 环境。

## 用户数据保护与可恢复方案

本次只读核验没有在默认 `%AppData%\Rime` 路径和常见 Rime 注册表位置发现现有配置。这个观察不证明用户没有其他位置或其他账号的 Rime 数据；未来运行测试前必须重新发现真实目录。没有因此创建、删除、清空或覆盖任何用户数据。

当前云端构建只操作一次性 hosted runner 的 checkout 和临时目录，不接触个人电脑的用户文件夹。下载 artifact 本身也不需要安装。为了保留可恢复性，本阶段不运行 artifact 内的 install/uninstall、WeaselSetup、WeaselDeployer 或 WeaselServer。

将来测试按以下顺序准备，不能把 `output/data` 复制到现有用户目录：

1. **优先独立 Windows 用户或虚拟机快照**。使用单独的测试数据目录和输入方案，避免与主账号的 TSF 注册、服务、配置和用户词库混用。仅把 ZIP 解压到另一个目录并不等于完全隔离。
2. 如果必须在主账号安装，先发现 `%AppData%\Rime`、`HKCU\SOFTWARE\Rime\Weasel` 的 `RimeUserDir`，以及 HKLM 的 32/64 位 `WeaselRoot`。记录当前安装版本、安装路径、输入语言 / TSF 注册信息，保留原安装器或完整原安装目录。
3. 在确认切换到其他输入法并让服务停止写词库后，将**整个真实用户目录**完整复制到仓库外的带日期备份目录，保留配置、build 缓存、userdb、sync 和 installation.yaml 等内容；不要只备份 YAML。对源与备份逐文件生成相对路径、长度、SHA256 清单并比对。
4. 另存相关注册表键导出、原版本安装器和安装目录清单。备份中含个人词库，不提交到 Git 或 Actions artifacts。使用复制式备份，不使用 robocopy /MIR 或任何会删除目标额外文件的同步操作。
5. 测试结束回滚时先停止测试服务并退出占用 DLL 的应用，保留测试后的完整数据副本，再恢复原二进制 / 注册信息和原用户目录，验证清单后恢复正常输入。先在隔离环境演练，避免直接对唯一副本覆盖恢复。

本阶段没有执行以上未来安装 / 回滚操作；恢复方案已经记录，输入法和用户数据保持原状。
