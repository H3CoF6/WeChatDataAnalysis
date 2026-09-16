# Linux 发布流程（x64）

Linux 与 Windows / macOS 一起发在同一个 tag Release 里，并且是**必需平台**：原生组件的 pin
没配齐时，`release.yml` 会直接失败，而不是静默少发一个平台。

产物形态刻意不做 AppImage / deb：Linux 走「用户级、免 root 的 `tar.gz` + `install.sh`」。

## 涉及的三个工作流

| 工作流 | 位置 | 作用 |
| --- | --- | --- |
| `linux-native-production.yml` | **WCDB**（私藏 producer 仓） | 手工 dispatch：构建 + 自检 + 把原生核心发成不可变 Release 资产，并把消费方要的 pin 打到 run summary |
| `linux-private-build.yml` | 本仓 | 可复用构建：下载 pin 产物 → 校验 → 编译 integrity → `dist:linux` → 打包校验 → 上传 |
| `release.yml` | 本仓 | `push tag v*` 触发；`build-linux-x64` 调用上面的可复用工作流，`publish-release` 汇总三个平台 |

## 操作顺序

1. **产原生核心**（在 WCDB producer 仓）：Actions → `Linux native production` → Run workflow。
   - `profile` = `source-public`（WCDA 只收这个 profile）
   - `build_id` 留空 → 取 `linux-x64-<run id>`；**重跑必须显式传新 ID**，已发布的 ID 不允许复用
   - `publish_release` = true
   - 需要 secret/var `WCE_ROOT_PUBLIC_KEY_HEX`（128 hex，P-256 **公钥**，不是私钥）
   - 结束后在 run summary 里复制那段 `WCE_LINUX_NATIVE_CORE_*` 变量清单
2. **配本仓变量**（Settings → Secrets and variables → Actions）：

   | 类型 | 名称 | 说明 |
   | --- | --- | --- |
   | variable | `WCE_LINUX_NATIVE_CORE_ARTIFACT_REPOSITORY` | producer 仓 `owner/repo`（= 运行工作流的那个仓） |
   | variable | `WCE_LINUX_NATIVE_CORE_ARTIFACT_DOWNLOAD_REPOSITORY` | 资产当前托管在哪；留空则同上 |
   | variable | `WCE_LINUX_NATIVE_CORE_ARTIFACT_RUN_ID` | producer 的 run id |
   | variable | `WCE_LINUX_NATIVE_CORE_ARTIFACT_SHA256` | 资产 tar.gz 的摘要 |
   | variable | `WCE_LINUX_NATIVE_CORE_SOURCE_REVISION` | 40 hex 的 WCDB revision |
   | variable | `WCE_LINUX_NATIVE_CORE_BUILD_ID` | 构建 ID |
   | variable | `WCE_LINUX_NATIVE_CORE_CLIENT_SHA256` / `..._BROKER_SHA256` | 可选的内容 pin；一旦填就必须与 manifest 一致 |
   | variable | `WCE_LINUX_INTEGRITY_SOURCE_REPOSITORY` / `..._SOURCE_REVISION` | 可选；默认取原生核心的仓库/revision（`private/wce_integrity` 在同一棵树里） |
   | secret | `WCE_LINUX_PRODUCER_READ_TOKEN` | 对 producer 仓有读权限的 token（资产下载 + 取 integrity 源码） |
3. **发版**：推 tag `v*`。tag 必须在 `origin/main` 上。

## 为什么可以不做代码签名

Linux 没有 Authenticode / codesign 的等价物，所以身份改成**内容哈希**，方向是单向的：

- broker 里编死了「随包 client 的哈希」；
- broker 自己的哈希由 manifest 声明、由安装方 pin。

`sha256(file) == pin(file)` 无解，所以「组件自带自身哈希」这种不可判定的方向被显式禁止：
`Test-LinuxNativeProductionArtifact.py` 与 `desktop/scripts/linux-native-core-packaging.cjs`
两头都断言了这一点。消费侧还会在打包后再哈希一次（`packaged ... differs from the reviewed
native artifact`），确保打包过程没有顺手重编。

导出完整性模块 `libwce_integrity.so` 由本仓在发布时用**一次性构建密钥**现编（语义等同于
官方的 `-GenerateEphemeralSigningKey`）：该密钥只用于导出物自身封签，权威封印是原生核心产出的
WES2 sidecar。之所以不在 producer 侧预编，是因为 `wce_integrity` 会把 Nuxt 的 CSS 编进去，
必须和当次 UI 构建同源。

## 会踩的坑

- **45 天有效期**：pin 的 manifest 固定 45 天窗口，到期后 `build-linux-x64` 会在校验阶段
  明确报 `build has reached its fixed expiration time`。到期必须重新产一份并更新 pin。
- **构建 ID 不可复用**：producer 在发布前会检查 `linux-native-<build-id>` 是否已存在，存在即拒绝。
- **校验失败就是失败**：`build-linux-x64` 不设 `continue-on-error`，`publish-release.needs` 包含它，
  所以 pin 缺失 / 摘要不符 / 哈希漂移都会让 release 停在半路而不是发出去。
- **产物名不能重**：Linux 用 `SHA256SUMS-linux.txt` / `release-provenance-linux.json`，
  避免与 Windows 的 `SHA256SUMS.txt` / `release-provenance.json` 在 `merge-multiple` 下载时互相覆盖。

## 已知缺口（发版前必须处理）

`desktop/src/native-core-runtime.cjs` 目前只认 `win32` / `darwin`：在 Linux 上
`resolveNativeCoreRuntimePolicy()` 会抛 `unsupported on platform: linux`，而它由
`startBackend()` 无保护调用 —— 即打出来的 Linux 包**启动后端就会失败**。
`desktop/tests/native-core-runtime.test.cjs` 在 Linux 上因此跑不过，所以没有进
`build-linux-x64` 的桌面测试门禁。在修好之前，Linux 的 release 只是「能出包」，不是「能用」。
