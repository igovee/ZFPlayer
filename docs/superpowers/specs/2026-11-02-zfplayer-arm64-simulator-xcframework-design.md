# ZFPlayer arm64 模拟器 xcframework

在这台安装了 Xcode 26.3 的机器上，用 `renzifeng/ZFPlayer` 的 `master` 提交 `1ede0b9fd80d52a1474201ee506285c9792acb46` 重打静态 xcframework，替换 `igovee/ZFPlayer` 里的二进制。产物要能被 Xcode 26 和 Xcode 27 链接，并在 Apple 芯片模拟器上提供 arm64。

这份文档先于二进制提交进入本仓库。随后的二进制提交只改 xcframework、bundle 和 podspec。tag `4.1.4` 指向二进制提交，因此打出的树里也包含这份文档。CocoaPods 不读取它。

## 决定

- 源码提交固定为 `1ede0b9fd80d52a1474201ee506285c9792acb46`（`master`，2023-01-06）。包含 tag `4.1.4` 之后的长按手势和 iOS 16 横屏窗口修复。
- 版本号保持 `4.1.4`。
- 只编译 Core、ControlView、AVPlayer，合成一个名为 `ZFPlayer` 的静态 framework。`ijkplayer` 不编译，不链接 `IJKMediaFramework`。
- 最低系统是 iOS 15.0。Xcode 26 和 Xcode 27 官方支持的部署版本从 iOS 15 起。
- 打包工具链是本机的 Xcode 26.3（`/Applications/Xcode.app`，`xcode-select -p` 已指向它）。使用 Xcode 26 的任一 26.x 发行版都符合要求。禁止改用 Xcode 27 打这个包。Xcode 27 产出的静态 `.o` 不保证能被 Xcode 26 的链接器接受。Xcode 26 产出的包，Xcode 26 和 Xcode 27 都能链。上传 App Store 时看的是编 App 的那套 Xcode，不是这个静态库的编译器版本。
- 架构：
  - `ios-arm64`：真机，仅 arm64，平台 iphoneos。
  - `ios-arm64_x86_64-simulator`：模拟器，同一个库里含 arm64 和 x86_64，平台 iphonesimulator。Apple 芯片模拟器使用其中的 arm64。
- 真机 arm64 和模拟器 arm64 分成 xcframework 里的两条库。
- `ZFPlayer.bundle` 以 CocoaPods `resources` 交给宿主 App。控制层用 `bundleForClass:` 在主 bundle 里找 `ZFPlayer.bundle`。
- 发布用的 `ZFPlayer.podspec.json` 只描述二进制。`source.git` 改为 `https://github.com/igovee/ZFPlayer.git`，tag 仍为 `4.1.4`。

## 仓库与机器

| 路径或仓库 | 本次用途 |
|---|---|
| `git@github.com:renzifeng/ZFPlayer.git` | 只读源码，检出指定提交 |
| `/Users/govee/Desktop/transfer/ZFPlayer`（`git@github.com:igovee/ZFPlayer.git`，分支 `main`） | 写入新的 xcframework、`ZFPlayer.bundle` 和 podspec |
| `/Users/govee/Desktop/transfer/zfplayer-pack` | 临时打包目录，位于 git 工作区之外。开始时整目录删除后重建 |

写这份文档时，`main` 是 `049b0d700a1ef870656cf76364e516e80d308fd9`，tag `4.1.4` 指向这次旧二进制。打包可以从包含这份文档的后续 `main` 开始。tag 要移动到二进制提交，而不是停在 `049b0d7`。工作区里未跟踪的 `未命名/` 不提交、不删除。

开始前确认：

- `xcodebuild -version` 的第一行匹配 `^Xcode 26`。不是 26 就停止。
- `xcode-select -p` 指向这套 Xcode 26。
- `ruby -e 'require "xcodeproj"'` 成功。
- 能克隆 `renzifeng/ZFPlayer`，并能推送 `igovee/ZFPlayer`。

打包过程产生的临时工程、Pods、DerivedData 不提交。

## 发布结果

`igovee/ZFPlayer` 的二进制提交只改这三样。`LICENSE` 和 `README.md` 不动。

- `ZFPlayer.xcframework`：一条真机库 `ios-arm64`（arm64），一条模拟器库 `ios-arm64_x86_64-simulator`（arm64 和 x86_64）。
- 仓库根目录的 `ZFPlayer.bundle`，由 podspec 的 `resources` 交给宿主 App。
- `ZFPlayer.podspec.json` 只描述二进制：版本 `4.1.4`，最低 iOS 15.0，`static_framework`，`vendored_frameworks` 指向 xcframework，`resources` 指向根目录的 bundle。`homepage` 仍是 `https://github.com/renzifeng/ZFPlayer`。`source.git` 是 `https://github.com/igovee/ZFPlayer.git`，tag 是 `4.1.4`。

发布 podspec 的全文：

```json
{
  "name": "ZFPlayer",
  "version": "4.1.4",
  "summary": "A good player made by renzifeng",
  "homepage": "https://github.com/renzifeng/ZFPlayer",
  "license": {
    "type": "MIT",
    "file": "LICENSE"
  },
  "authors": {
    "renzifeng": "zifeng1300@gmail.com"
  },
  "source": {
    "git": "https://github.com/igovee/ZFPlayer.git",
    "tag": "4.1.4"
  },
  "platforms": {
    "ios": "15.0"
  },
  "requires_arc": true,
  "static_framework": true,
  "vendored_frameworks": "ZFPlayer.xcframework",
  "resources": "ZFPlayer.bundle",
  "frameworks": [
    "UIKit",
    "AVFoundation",
    "AVKit",
    "MediaPlayer",
    "CoreTelephony",
    "SystemConfiguration"
  ]
}
```

宿主按 tag `4.1.4` 安装会拿到新包，因为这个 tag 会改指到新提交。已经拉过旧 tag 的人需要重新拉取。宿主工程自己写了 `EXCLUDED_ARCHS[sdk=iphonesimulator*] = arm64` 时，新包里的模拟器 arm64 仍然不会被链上。那个设置在宿主工程里。

## 构建

在 `/Users/govee/Desktop/transfer/zfplayer-pack` 执行。这个目录在 `igovee/ZFPlayer` 仓库外面。

```bash
set -euo pipefail

ROOT="/Users/govee/Desktop/transfer/zfplayer-pack"
SRC="$ROOT/src"
PACK="$ROOT/pack"
rm -rf "$ROOT"
mkdir -p "$SRC" "$PACK"

git clone https://github.com/renzifeng/ZFPlayer.git "$SRC"
git -C "$SRC" checkout 1ede0b9fd80d52a1474201ee506285c9792acb46
test "$(git -C "$SRC" rev-parse HEAD)" = "1ede0b9fd80d52a1474201ee506285c9792acb46"

xcodebuild -version | head -1 | grep -q '^Xcode 26'
```

用下面的内容覆盖 `$SRC/ZFPlayer.podspec`。这份 spec 只用于编译，不提交回上游，也不作为 `igovee/ZFPlayer` 的发布 spec。

```ruby
Pod::Spec.new do |s|
  s.name             = 'ZFPlayer'
  s.version          = '4.1.4'
  s.summary          = 'A good player made by renzifeng'
  s.homepage         = 'https://github.com/renzifeng/ZFPlayer'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'renzifeng' => 'zifeng1300@gmail.com' }
  s.source           = { :git => 'https://github.com/renzifeng/ZFPlayer.git', :tag => s.version.to_s }
  s.ios.deployment_target = '15.0'
  s.requires_arc = true
  s.static_framework = true
  s.source_files = 'ZFPlayer/Classes/Core/**/*.{h,m}',
                   'ZFPlayer/Classes/ControlView/**/*.{h,m}',
                   'ZFPlayer/Classes/AVPlayer/**/*.{h,m}'
  s.public_header_files = 'ZFPlayer/Classes/Core/**/*.h',
                          'ZFPlayer/Classes/ControlView/**/*.h',
                          'ZFPlayer/Classes/AVPlayer/**/*.h'
  s.resources = 'ZFPlayer/Classes/ControlView/ZFPlayer.bundle'
  s.frameworks = 'UIKit', 'AVFoundation', 'AVKit', 'MediaPlayer', 'CoreTelephony', 'SystemConfiguration'
  s.pod_target_xcconfig = {
    'DEFINES_MODULE' => 'YES',
    'IPHONEOS_DEPLOYMENT_TARGET' => '15.0',
    'ENABLE_BITCODE' => 'NO'
  }
end
```

下面的命令和上一段在同一个 shell 里执行，`$ROOT`、`$SRC`、`$PACK` 仍然有效。

在 `$PACK` 生成一个空的 iOS App 工程，只为了让 CocoaPods 生成 `Pods.xcodeproj`：

```bash
cd "$PACK"
ruby <<'RUBY'
require 'xcodeproj'
project = Xcodeproj::Project.new('Pack.xcodeproj')
project.new_target(:application, 'Pack', :ios, '15.0')
project.save
RUBY

cat > Podfile <<'EOF'
platform :ios, '15.0'
use_frameworks! :linkage => :static

target 'Pack' do
  pod 'ZFPlayer', :path => '../src'
end
EOF

pod install
```

`pod install` 之后，Pods 工程里必须有且只有一个名为 `ZFPlayer` 的 target。若出现 `ZFPlayer-Core`、`ZFPlayer-ControlView`、`ZFPlayer-ijkplayer`，停止。

两次 Release 编译都显式传 `IPHONEOS_DEPLOYMENT_TARGET=15.0`、`ONLY_ACTIVE_ARCH=NO`、`EXCLUDED_ARCHS=`、`CODE_SIGNING_ALLOWED=NO`。

```bash
cd "$PACK"

xcodebuild build \
  -project Pods/Pods.xcodeproj \
  -target ZFPlayer \
  -configuration Release \
  -sdk iphoneos \
  -derivedDataPath "$ROOT/dd-ios" \
  ARCHS=arm64 \
  ONLY_ACTIVE_ARCH=NO \
  IPHONEOS_DEPLOYMENT_TARGET=15.0 \
  EXCLUDED_ARCHS= \
  CODE_SIGNING_ALLOWED=NO

xcodebuild build \
  -project Pods/Pods.xcodeproj \
  -target ZFPlayer \
  -configuration Release \
  -sdk iphonesimulator \
  -derivedDataPath "$ROOT/dd-sim" \
  ARCHS='arm64 x86_64' \
  ONLY_ACTIVE_ARCH=NO \
  IPHONEOS_DEPLOYMENT_TARGET=15.0 \
  EXCLUDED_ARCHS= \
  CODE_SIGNING_ALLOWED=NO
```

每个 SDK 的产物目录里必须恰好有一个 `ZFPlayer.framework`。找到零个或两个以上都停止。真机来自 `Release-iphoneos`，模拟器来自 `Release-iphonesimulator`。

```bash
ios_hits=$(find "$ROOT/dd-ios/Build/Products/Release-iphoneos" -name ZFPlayer.framework -type d)
sim_hits=$(find "$ROOT/dd-sim/Build/Products/Release-iphonesimulator" -name ZFPlayer.framework -type d)
test "$(printf '%s\n' "$ios_hits" | sed '/^$/d' | wc -l | tr -d ' ')" = "1"
test "$(printf '%s\n' "$sim_hits" | sed '/^$/d' | wc -l | tr -d ' ')" = "1"
IOS_FW="$ios_hits"
SIM_FW="$sim_hits"

rm -rf "$ROOT/ZFPlayer.xcframework"
xcodebuild -create-xcframework \
  -framework "$IOS_FW" \
  -framework "$SIM_FW" \
  -output "$ROOT/ZFPlayer.xcframework"

cp -R "$SRC/ZFPlayer/Classes/ControlView/ZFPlayer.bundle" "$ROOT/ZFPlayer.bundle"
```

资源以源码目录里的 `ZFPlayer.bundle` 为准。

## 替换前校验

下面 8 项有任何一项失败就停止，不覆盖 `/Users/govee/Desktop/transfer/ZFPlayer` 里的旧 xcframework，也不提交、不推送、不移动 tag。

1. `lipo -info` 真机二进制只包含 `arm64`。
2. `lipo -info` 模拟器二进制同时包含 `arm64` 和 `x86_64`。
3. xcframework 的 `Info.plist` 有且只有两条 `AvailableLibraries`：
   - `LibraryIdentifier` 为 `ios-arm64`，`SupportedPlatform` 为 `ios`，`SupportedArchitectures` 为 `arm64`，没有 `SupportedPlatformVariant`。
   - `LibraryIdentifier` 为 `ios-arm64_x86_64-simulator`，`SupportedPlatform` 为 `ios`，`SupportedPlatformVariant` 为 `simulator`，`SupportedArchitectures` 同时含 `arm64` 和 `x86_64`。
4. 两个 framework 都有 `Modules/module.modulemap`、`Headers/ZFPlayer.h`、`Headers/ZFPlayerControlView.h`、`Headers/ZFAVPlayerManager.h`。
5. 从真机库和模拟器库各抽出一个 `.o`，`vtool -show-build` 的结果是：
   - 真机：`platform IOS`，`minos 15.0`。
   - 模拟器 arm64 和模拟器 x86_64：`platform IOSSIMULATOR`，`minos 15.0`。
   - `minos` 不得是 26.x 或 27.x。模拟器对象的 platform 不得是 `IOS`。
6. `ar -t` 两个二进制，输出里没有 `IJK` 或 `ZFIJKPlayerManager`。
7. `$ROOT/ZFPlayer.bundle/ZFPlayer_play@2x.png` 存在。
8. 即将写入的发布 podspec 文本里没有 `EXCLUDED_ARCHS`。

三个架构分三个空目录抽出 `.o`，避免读到上一次解出来的文件。模拟器先薄拆再解包；真机库本身只有 arm64，直接解包。每个目录取第一个名字不以 `__.` 开头的 `.o`，对它执行 `vtool -show-build`。

```bash
lipo "$SIM_FW/ZFPlayer" -thin arm64 -output "$ROOT/sim-arm64.a"
lipo "$SIM_FW/ZFPlayer" -thin x86_64 -output "$ROOT/sim-x86_64.a"

inspect() {
  local archive="$1" dest="$2"
  rm -rf "$dest"
  mkdir -p "$dest"
  (cd "$dest" && ar -x "$archive")
  local obj
  obj=$(find "$dest" -name '*.o' ! -name '__.*' | head -1)
  test -n "$obj"
  vtool -show-build "$obj"
}

inspect "$IOS_FW/ZFPlayer" "$ROOT/objs-ios"
inspect "$ROOT/sim-arm64.a" "$ROOT/objs-sim-arm64"
inspect "$ROOT/sim-x86_64.a" "$ROOT/objs-sim-x86_64"
```

`$ROOT/objs-ios` 的结果必须是 `platform IOS`、`minos 15.0`。另外两个目录必须是 `platform IOSSIMULATOR`、`minos 15.0`。上面的命令负责解包并打印 `vtool`。平台或 `minos` 不符合时，以非零状态退出。`lipo -info`、`ar -t`、头文件和 `Info.plist` 也一样：打印出来还不算通过，必须按第 1 到第 8 项核对，不符合就退出。

## 写入、提交、推送

校验通过后，在 `/Users/govee/Desktop/transfer/ZFPlayer` 的 `main` 上：

- 用 `$ROOT/ZFPlayer.xcframework` 整个替换仓库根目录的 `ZFPlayer.xcframework`。
- 把 `$ROOT/ZFPlayer.bundle` 放到仓库根目录。
- 用上面的 JSON 替换 `ZFPlayer.podspec.json`。

`未命名/` 不进入提交。提交说明这一句：

```
[Modify] 用 master 1ede0b9 重打 xcframework，补上 arm64 模拟器，并去掉模拟器 arm64 排除。
```

推送 `main`。`main` 使用普通推送。

## 移动 tag

`igovee/ZFPlayer` 上已经有 tag `4.1.4`，指向旧二进制。新提交推上去之后，按 tag 安装仍然拿到旧包，直到这个 tag 改指新提交。

在确认新提交已经在 `origin/main` 上之后：

```bash
git fetch origin
git tag -d 4.1.4
git push origin :refs/tags/4.1.4
git tag 4.1.4 origin/main
git push origin 4.1.4
```

这会改写已经发出去的 tag。新 tag 是轻量 tag，指向当时的 `origin/main`。已经按 `4.1.4` 拉过旧包的人需要重新拉取这个 tag。

## 范围外

- 不修改 `renzifeng/ZFPlayer`。
- 不把 `ijkplayer` 打进包。
- 不用 Xcode 27 编译。
- 不改 ZFPlayer 的播放、手势、转屏逻辑。`1ede0b9` 相对上游 tag `4.1.4` 的提交原样进入二进制。
- 不改宿主工程里的 `EXCLUDED_ARCHS`。
- 不提交、不删除工作区里的 `未命名/`。

## 完成标准

- 替换前的 8 项校验都通过。
- `origin/main` 上有新的 `ZFPlayer.xcframework`、根目录 `ZFPlayer.bundle` 和上面的 `ZFPlayer.podspec.json`。
- tag `4.1.4` 指向这次二进制提交。
