# iOS 孩子端：构建与安装

本工程是「家人位置共享」的 iOS 原生孩子端（24/7 后台定位 + 加密上报 + 断网缓存 + 自动恢复）。
**构建 iOS App 必须用 macOS + Xcode**（苹果硬性要求，Windows 无法生成 .ipa）。

代码已全部完成并通过跨平台一致性设计（`ios/tools/verify_vectors.swift` 会与 Android 生成的
`docs/vectors.json` 逐字节校验：PBKDF2 密钥、AES-GCM 密文、topic 指纹）。

## 三条构建路径（按省事程度排序）

### 路径一：GitHub Actions 云构建（全自动，只需注册一次 GitHub）

1. 注册 GitHub 账号（免费，邮箱即可）：https://github.com/signup
2. 新建一个 **Public** 仓库（不要选 Private，Public 才有免费 macOS 云 Mac）
3. 把本文件夹（`family-locator` 整个项目）上传到该仓库（网页上传 zip 不行，用：
   - 网页端「Add file → Upload files」把文件拖进去；或者
   - 命令行 `git push`）
4. 打开仓库页面 → Actions → 选 `ios-build` → Run workflow → 等约 10 分钟
5. 构建完成后在该次运行的 Artifacts 里下载 `KidLocator-ipa`（得到 `KidLocator.ipa`，未签名）
6. 用 **AltStore 把 ipa 装到 iPhone**（见下方「AltStore 安装」）

### 路径二：云 Mac 租用 + 远程操作（我可以全程替你操作）

1. 租用一台云 Mac（MacInCloud 按 小时计费 约 $1-2/小时，或月付）：
   - https://www.macincloud.com（选 Pay-As-You-Go，支持网页浏览器远程连接）
2. 把云 Mac 的**浏览器连接地址 + 登录账号密码**发给我
3. 我通过浏览器远程操作：装 Xcode 命令行工具 → 校验向量 → 编译出 `KidLocator.ipa`
4. 你在 Windows 上用 AltStore 把 ipa 装进 iPhone（见下方）

### 路径三：借一次 Mac（有 Mac 朋友的话）

1. Mac 上安装 Xcode（App Store，免费）
2. 终端执行：
   ```bash
   brew install xcodegen
   cd ios && xcodegen generate
   xcodebuild -project KidLocator.xcodeproj -scheme KidLocator \
     -configuration Release -sdk iphoneos -archivePath build/K.xcarchive archive \
     CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO
   # 打包 ipa
   mkdir -p build/Payload && cp -R build/K.xcarchive/Products/Applications/KidLocator.app build/Payload/
   cd build && zip -r -y KidLocator.ipa Payload
   ```
3. 编译前先跑向量校验：
   ```bash
   swiftc KidLocator/Crypto.swift KidLocator/TopicUtils.swift tools/verify_vectors.swift -o /tmp/verify
   /tmp/verify ../docs/vectors.json     # 必须输出 ALL VECTORS PASSED
   ```

## AltStore 安装（Windows 即可，无需 Mac，免费苹果 ID）

AltStore 用你的**免费 Apple ID** 给未签名 ipa 重签名并安装到 iPhone：

1. Windows 上安装：https://altstore.io → AltServer (Windows)
2. iPhone 用数据线连电脑，信任电脑；iTunes 安装（Windows 商店版需选「同步」）
3. 右下角托盘 AltServer → Install AltStore → 选你的 iPhone → 输入 Apple ID + 密码
   （苹果 ID 需在 https://appleid.apple.com 开启双重认证）
4. iPhone 上打开 AltStore → 点左上角「+」→ 选择 `KidLocator.ipa` → 安装
5. 设置 → 通用 → VPN与设备管理 → 信任你的 Apple ID 开发者描述文件

**免费账号限制**：签名 7 天有效；AltStore 在 iPhone 与电脑同 WiFi 时会自动后台刷新签名，
一次都不用管。若嫌麻烦可买苹果开发者账号（$99/年）改用 TestFlight，一年装一次。

## 首次使用（孩子的 iPhone 上）

1. 打开「位置共享」→ 输入配对码（与家长端相同）→ 开始共享
2. 定位权限选**「始终允许」**（24/7 后台必需；只给「使用期间」则锁屏后停）
3. 顶部状态栏出现定位箭头 = 正在上报
4. 建议设置 → 电池 → 关闭低电量模式（省电模式下 iOS 可能降频定位）

## 与家长端（Android）的对应关系

| 孩子端 iOS | 家长端 Android |
|---|---|
| 配对码 | 同一配对码 |
| 每 30 秒/移动 10 米上报 | 15 秒轮询 + retained |
| AES-GCM 加密（同 Java 向量） | 解密显示 |
| 断网缓存 500 条，恢复补发 | — |
| 杀进程/重启后由系统恢复（显著位置变化会自动唤醒 App） | — |
