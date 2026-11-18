# 个人所得税 · iOS 壳工程

把 `https://files.flongli.top` 这站包成 iOS App。**不需要 Mac**——用 GitHub Actions 的 macOS runner 编译。

产物是**未签名 IPA**，配合 TrollStore（巨魔）安装；没有巨魔就用全能签加证书签。

---

## 一、它做什么

`www/index.html` 是个 WKWebView 壳：

- 显示启动图（`start.png`）
- 建一个 iframe 加载 `https://files.flongli.top/mine.html?in_app=1&ios_shell=1`
- UA 追加 ` TaxPlatformCordovaApp/1 (iOS)`，站点里那些 `isCordovaApp()` 分支会亮
- postMessage 桥：外部链接、剪贴板、设备信息
- 状态栏透明叠加

`config.xml` 里锁死了只允许加载 `files.flongli.top`：

```xml
<access origin="https://files.flongli.top" />
<allow-navigation href="https://files.flongli.top/*" />
```

---

## 二、推送并构建

### 1. 建仓库

在 GitHub 新建一个空仓库（private 也行，macOS runner 每月有 2000 分钟免费额度，一次构建约 5–8 分钟）。

### 2. 推代码

```bash
cd ios-shell
git init
git add -A
git commit -m "init ios shell"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

### 3. 等 Actions

push 上去会自动触发 `.github/workflows/build-ios.yml`。也可以在 Actions 页面点 **Run workflow** 手动跑。

跑完在 **Artifacts** 里下载 `geshui-ios-ipa`，解压得到 `geshui-ios-unsigned.ipa`。

### 4. 构建过程

```
macos-14 runner
  ├─ Xcode 15.4
  ├─ npm i -g cordova@12
  ├─ cordova platform add ios@7
  ├─ cordova build ios --release --device   (CODE_SIGNING_ALLOWED=NO)
  ├─ ldid -S 伪签名（加 platform-application entitlement）
  └─ zip → geshui-ios-unsigned.ipa
```

---

## 三、安装

### 方式 A：TrollStore（推荐）

1. 把 IPA 传到手机（AirDrop / 文件 App / 局域网）
2. TrollStore → 右上角 **+** → 选 IPA
3. 装完桌面出现「个人所得税」

不需要证书，不掉签。前提是设备/系统在 TrollStore 支持范围（iOS 14.0–16.6.1、17.0 等，看设备）。

### 方式 B：全能签

1. 全能签 → 导入 IPA
2. 选一个可用证书（p12 + mobileprovision）
3. 签名安装

免费 Apple ID 证书 7 天过期，需要重签。

### 方式 C：Sideloadly（Windows）

装 iTunes + Sideloadly，登录 Apple ID 直接签，同样 7 天。

---

## 四、要改的地方

| 想改什么 | 改哪里 |
|---|---|
| 加载的域名 | `www/index.html` 的 `APP_ORIGIN` 和 `ALLOWED_HOST`，同时改 `config.xml` 的 `<access>` / `<allow-navigation>` |
| 启动图 | 替换 `www/start.png` |
| 图标 | 替换 `res/icon/icon-1024.png`（1024×1024 PNG） |
| 显示名 | `config.xml` 里的 `CFBundleDisplayName` |
| Bundle ID | `config.xml` 顶部的 `id`（默认 `com.testplatform.app`） |
| UA 标识 | `config.xml` 的 `AppendUserAgent` |

改完 push，Actions 重新出包。

---

## 五、备选：完全本地（不碰云）

WSL2 + Ubuntu 里装 Theos：

```bash
export THEOS=~/theos
git clone --recursive https://github.com/theos/theos.git $THEOS
# 从 https://github.com/theos/sdks 下 iPhoneOS16.5.sdk 放到 $THEOS/sdks/
```

然后用 clang + ld64 + ldid 编译 `.app` 再 zip 成 IPA。可行，但工具链要自己踩，不如 Actions 省事。

---

## 六、已知限制

- **没有原生相册写入**：站里「添加到相册」在 iOS 壳内会失败（回执 `ios_no_native_save`），用户长按图片保存即可。要做的话得加 `cordova-plugin-file` + `@capacitor/filesystem` 之类，另说。
- **推送**：没接 APNs。
- **未签名 IPA**：只能用 TrollStore 或自己签，不能直接双击装。
