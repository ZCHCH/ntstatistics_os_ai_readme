# NTStatisticsOS SDK – AI 集成说明（中文版）

## 目的

本说明文档用于指导 **AI 自动完成**  
**NTStatisticsOS iOS 统计 SDK（海外版 2.x.x）** 的集成、初始化与验证。

AI 必须严格按顺序执行，不可跳步。AI 需要基于用户提示词（如“访问文档，帮我集成统计SDK、ATT、AppsFlyer(或者Adjust)、数数”）动态确定集成内容：
- **统计SDK**：基础集成，必须。
- **ATT**：仅用于启用 IDFA 支持（配置 config.idfaEnable = true + 隐私文件配置），**不涉及任何 ATT 权限请求代码**（如 ATTrackingManager.requestTrackingAuthorization），SDK 内部处理权限逻辑。
- **AppsFlyer** 或 **Adjust**：归因平台（可选，二选一，不可同时；如果提示词未指定，则不集成）。
- **数数**（Thinking Analytics）：数据分析（可选）。

通过提示词解析集成项，生成相应代码和配置。AI 需要判断当前项目语言是 Swift 还是 Objective-C（基于用户指定或项目上下文，默认 Objective-C）。如果为 Swift，直接使用 Swift 语法调用 SDK 方法（无需桥接头文件）；如果为 Objective-C，使用 OC 语法。所有代码确保兼容，不要使用桥接。

此外，判断项目生命周期管理方式：
- **传统 AppDelegate**（默认或旧项目）
- **SceneDelegate**（iOS 13+ 项目，如果项目中存在 SceneDelegate.swift / SceneDelegate.m 文件，则优先使用）
- **SwiftUI**（如果项目使用 @main App struct + scenePhase）

**AI 必须先判断项目结构**：
- 检查是否存在 SceneDelegate 文件 → 使用 SceneDelegate
- 检查是否为 SwiftUI 项目（存在 @main App struct） → 使用 scenePhase
- 否则 → 使用 AppDelegate

---

## 必须输入参数

在执行前，参数基于提示词动态确定，但无需在提示词中显式输入。AI 在生成代码时，使用以下占位符作为默认值，用户需手动替换为实际值：

| 参数名 | 说明 | 默认占位符 |
|--------|------|------------|
| `APP_ID` | 项目 AppID | “你的appid” |
| `APP_KEY` | 项目 AppKey | “你的appkey” |
| `ATTRIBUTION` | `AppsFlyer` 或 `Adjust`（如果提示词指定） | 无（不集成归因） |
| `USE_THINKING` | `true / false`（如果提示词包含“数数”则 true） | false |
| `MAIN_URL` | 转发域名 | “你的mainurl” |
| `APPLE_APP_ID` | App Store 的 AppleAppID（AppsFlyer 需要） | “你的appleappid” |
| `APPSFLYER_KEY` | AppsFlyer DevKey（AppsFlyer 需要） | “你的appsflyerkey” |
| `THINK_APP_ID` | 数数 AppID（USE_THINKING=true 需要） | “你的thinkappid” |
| `THINK_SERVER` | 数数转发域名（USE_THINKING=true 需要） | “你的thinkserver” |
| `ADJUST_TOKEN` | Adjust Token（Adjust 需要） | “你的adjusttoken” |

如果提示词包含“ATT”，则自动开启 IDFA 支持（仅配置 idfaEnable 和隐私文件）。

---

## Step 1：配置 Podfile

AI 先检查本地 Podfile 是否存在私有源索引：
- 执行命令：`pod repo list` 检查是否包含 `NTZhongTaiSpec`。
- 如果没有，添加私有源：执行 `pod repo add NTZhongTaiSpec http://coding.nineton.cn/zhongtai/NTZhongTaiSpec.git` 并 `pod repo update`。

在 Podfile 顶部添加（如果未存在）：

```ruby
source 'http://coding.nineton.cn/zhongtai/NTZhongTaiSpec.git'
```

添加基础 SDK：

```ruby
pod 'NTStatisticOSSDK'
```

如果提示词包含“数数”（`USE_THINKING = true`）：

```ruby
pod 'NTStatisticOSSDK/Think'
```

如果 `ATTRIBUTION = AppsFlyer`：

```ruby
pod 'NTStatisticOSSDK/AppsFlyerOS'
```

如果 `ATTRIBUTION = Adjust`：

```ruby
pod 'NTStatisticOSSDK/AdjustOS'
pod 'GoogleAdsOnDeviceConversion', '3.0.0'
```

如果无归因（ATTRIBUTION 未指定），则不添加归因相关 pod。

执行：

```bash
pod install --repo-update
```

---

## Step 2：隐私协议控制（必须）

在合适的生命周期入口文件中导入头文件（Swift：import NTStatisticsOSSDK；OC：#import <NTStatisticsOS/NTStatisticsOS.h>）。

应用启动时（默认不同意）并注册回调：

**AppDelegate 版本（didFinishLaunchingWithOptions）**：

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    NTStatisticsOS.nt_setAgreement(false)
    NTStatisticsOS.nt_agreementCallBack {
        self.initStatisticsSDK()  // 初始化
    }
    return true
}
```

**SceneDelegate 版本（scene willConnectTo）**：

```swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
    NTStatisticsOS.nt_setAgreement(false)
    NTStatisticsOS.nt_agreementCallBack {
        self.initStatisticsSDK()  // 初始化
    }
}
```

**SwiftUI 版本（在 @main App struct 的 init() 或 ContentView onAppear）**：

```swift
init() {
    NTStatisticsOS.nt_setAgreement(false)
    NTStatisticsOS.nt_agreementCallBack {
        self.initStatisticsSDK()
    }
}
```

用户同意隐私协议后（业务方调用）：

```swift
NTStatisticsOS.nt_setAgreement(true)
```

规则：
- SDK 不保存隐私状态
- 由宿主 App 控制
- 初始化必须在同意回调内执行

---

## Step 3：SDK 初始化

推荐封装私有方法，在隐私同意回调内调用。

**Swift 版本（所有项目通用）**：

```swift
private func initStatisticsSDK() {
    NTStatisticsOS.nt_init { config in
        config.mainURL = "你的mainurl"
        config.app_id = "你的appid"
        config.app_key = "你的appkey"
        config.channel = NTChannelType.NTAPPStore
        config.idfaEnable = false  // 默认 false，如果提示词包含“ATT”则 true

        // 动态配置 source（AppsFlyer 或 Adjust，二选一；如果无，则不设置）
        if "<ATTRIBUTION>" == "AppsFlyer" {
            config.source = [NSNumber(value: NTAttributionSource.appsFlyer.rawValue)]
            #if DEBUG
            config.appsFlyerTest = true
            #else
            config.appsFlyerTest = false
            #endif
            config.appleAppID = "你的appleappid"
            config.appsFlyerDevKey = "你的appsflyerkey"
        } else if "<ATTRIBUTION>" == "Adjust" {
            config.source = [NSNumber(value: NTAttributionSource.adJust.rawValue)]
            #if DEBUG
            config.adJustTest = true
            #else
            config.adJustTest = false
            #endif
            config.adJustToken = "你的adjusttoken"
            config.adJustLoglevel = 1
        }

        // 如果 USE_THINKING = true
        if <USE_THINKING> {
            config.thinkAppID = "你的thinkappid"
            config.thinkServe = "你的thinkserver"
            #if DEBUG
            config.thinkTest = true
            #else
            config.thinkTest = false
            #endif
        }

        return config
    } complete: { success in
        print("统计SDK初始化完成: \(success)")
    }
}
```

**Objective-C 版本**（保持原有）：

```objc
[NTStatisticsOS nt_initWithConfig:^NTStatisticsOSConfig *(NTStatisticsOSConfig *config) {
    config.MainURL = @"你的mainurl";
    config.app_id = @"你的appid";
    config.app_key = @"你的appkey";
    config.channel = NTAPPStore;
    config.idfaEnable = NO;  // 默认 NO，如果提示词包含“ATT”则 YES

    // 动态配置 source（AppsFlyer 或 Adjust，二选一；如果无，则不设置）
    if ([@"<ATTRIBUTION>" isEqualToString:@"AppsFlyer"]) {
        config.source = @[@(NTAppsFlyer)];
        #ifdef DEBUG
        config.appsFlyerTest = YES;
        #else
        config.appsFlyerTest = NO;
        #endif
        config.appleAppID = @"你的appleappid";
        config.appsFlyerDevKey = @"你的appsflyerkey";
    } else if ([@"<ATTRIBUTION>" isEqualToString:@"Adjust"]) {
        config.source = @[@(NTAdjust)];
        #ifdef DEBUG
        config.adJustTest = YES;
        #else
        config.adJustTest = NO;
        #endif
        config.adJustToken = @"你的adjusttoken";
        config.adJustLoglevel = 1;
    }

    // 如果 USE_THINKING = true
    if (<USE_THINKING>) {
        config.thinkAppID = @"你的thinkappid";
        config.thinkServe = @"你的thinkserver";
        #ifdef DEBUG
        config.thinkTest = YES;
        #else
        config.thinkTest = NO;
        #endif
    }

    return config;
} completeBlock:^(BOOL success) {
}];
```

---

## Step 4：前后台生命周期

在合适的文件中添加生命周期方法（**不允许加条件判断**）。

**AppDelegate 版本**：

```swift
func applicationDidBecomeActive(_ application: UIApplication) {
    NTStatisticsOS.nt_didBecomeActive()
}

func applicationWillResignActive(_ application: UIApplication) {
    NTStatisticsOS.nt_willResignActive()
}
```

**SceneDelegate 版本**（如果项目存在 SceneDelegate 文件，则优先使用）：

```swift
func sceneDidBecomeActive(_ scene: UIScene) {
    NTStatisticsOS.nt_didBecomeActive()
}

func sceneWillResignActive(_ scene: UIScene) {
    NTStatisticsOS.nt_willResignActive()
}
```

**SwiftUI 版本**（使用 scenePhase）：

```swift
import SwiftUI
import NTStatisticsOSSDK

@main
struct YourApp: App {
    @Environment(\.scenePhase) private var scenePhase

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .onChange(of: scenePhase) { newPhase in
            switch newPhase {
            case .active:
                NTStatisticsOS.nt_didBecomeActive()
            case .inactive:
                NTStatisticsOS.nt_willResignActive()
            default:
                break
            }
        }
    }
}
```

规则：
- AI 必须判断项目结构：
  - 若存在 SceneDelegate 文件 → 前后台代码添加到 SceneDelegate（sceneDidBecomeActive / sceneWillResignActive）
  - 若为 SwiftUI 项目 → 使用 scenePhase
  - 否则 → 添加到 AppDelegate
- SDK 会在未同意隐私前缓存事件

---

## Step 5：设备唯一标识 ZT

在当前类中导入头文件（如上）。

获取方式：

**Swift 版本（包括 SwiftUI）：**

```swift
let zt = NTStatisticsOS.userIdentifier()
```

**Objective-C 版本：**

```objc
NSString *zt = [NTStatisticsOS userIdentifier];
```

规则：
- 禁止缓存
- 每次创建订单时实时获取
- 必须上传给后端

---

## Step 6：用户与会员状态

在当前类中导入头文件（如上）。

设置用户 ID：

**Swift 版本（包括 SwiftUI）：**

```swift
NTStatisticsOS.setUserId("你的userid")
```

**Objective-C 版本：**

```objc
[NTStatisticsOS setUserId:@"你的userid"];
```

设置会员状态：

**Swift 版本（包括 SwiftUI）：**

```swift
NTStatisticsOS.setReward(true)
```

**Objective-C 版本：**

```objc
[NTStatisticsOS setReward:YES];
```

调用示例：注释说明在用户状态变更时调用

---

## Step 7：归因监听

在当前类中导入相应头文件（Swift：import NTAppsFlyerOS 或 import NTAdjustOS；OC：#import <NTAppsFlyerOS/NTAppsFlyerOS.h> 或 #import <NTAdjustOS/NTAdjustOS.h>）。仅如果有归因。

对于 SwiftUI 项目：在主 App struct 或相关 View 中添加观察者（使用 NotificationCenter.default.addObserver），并记得在 deinit 或 onDisappear 中移除观察者。

### AppsFlyer（如果 ATTRIBUTION = AppsFlyer）

监听通知名：Objective-C中**NTAFAttributionNotification**，Swift和SwiftUI中使用**Notification.Name.NTAFAttribution**

**Swift 版本（包括 SwiftUI）：**

```swift
NotificationCenter.default.addObserver(
    self,
    selector: #selector(attributionNotification(_:)),
    name: Notification.Name.NTAFAttribution,
    object: nil
)
```

**Objective-C 版本：**

```objc
[[NSNotificationCenter defaultCenter] addObserver:self
    selector:@selector(attributionNotification:)
    name:NTAFAttributionNotification
    object:nil];
```

回调处理（示例）：

- userInfo 包含：
  - 成功：@"status":@"success", @"result": conversionInfo（需原样透传）, @"attribution": ..., @"source":@"af"
  - 失败：@"status":@"fail", @"error": error, @"source":@"af"

**必须**：无论成功或失败，如果有 @"result"，原样透传给业务后端。

### Adjust（如果 ATTRIBUTION = Adjust）

监听通知名：Objective-C中**NTAttributionNotification**，Swift和SwiftUI中使用**Notification.Name.NTAttribution**

**Swift 版本（包括 SwiftUI）：**

```swift
NotificationCenter.default.addObserver(
    self,
    selector: #selector(attributionNotification(_:)),
    name: Notification.Name.NTAttribution,
    object: nil
)
```

**Objective-C 版本：**

```objc
[[NSNotificationCenter defaultCenter] addObserver:self
    selector:@selector(attributionNotification:)
    name:NTAttributionNotification
    object:nil];
```

回调处理（示例）：

- userInfo 包含：
  - @"result": [attribution jsonResponse]（需原样透传）
  - @"attribution": ...
  - @"source": @"adjust"

监听回调方法：**attributionNotification**
**必须**在@"result"为空时调用主动获取方式：

///注意：除了监听通知外，还可以主动调用以下方法获取归因信息  
///可以在需要归因之前多次调用此方法（例如登录前、支付前等关键节点），以确保获取到最新归因数据  
///result 回调中的字典即为归因信息，其中 @"result" 需要原样透传给业务后端

**Objective-C 版本**：

```objc
#import <NTAdjustOS/NTSTAdJustLibTool.h>

- (void)attributionNotification:(NSNotification *)notification {
    if ([notification.userInfo objectForKey:@"result"]) {
        // TODO: 将 result 上传给业务后端 让后端取归因渠道/平台
        NSLog(@"Adjust 归因信息，需透传后端: %@",[notification.userInfo objectForKey:@"result"]);
    } else {
        [NTSTAdJustLibTool AttributionResult:^(NSDictionary *dic) {
            /// 可以在需要归因之前多次调用此方法（例如登录前、支付前等关键节点），以确保获取到最新归因数据
            /// dic 即为归因信息，原样透传给业务后端 让后端取归因渠道/平台
        }];
    }
}
```

**Swift 版本**（需导入对应模块）：

```swift
import NTAdjustOS

@objc private func attributionNotification(_ notification: Notification) {
        guard let userInfo = notification.userInfo else { return }
        // 必须：将 result 原样透传给业务后端 让后端取归因渠道/平台
        if let result = userInfo["result"] {
            // TODO: 将 result 上传给业务后端 让后端取归因渠道/平台
            print("Adjust 归因信息，需透传后端: \(result)")
        } else {
            NTSTAdJustLibTool.attributionResult { dic in
                /// 可以在需要归因之前多次调用此方法（例如登录前、支付前等关键节点），以确保获取到最新归因数据  
                 /// dic 即为归因信息，原样透传给业务后端 让后端取归因渠道/平台
            }
        }
    }

```

**必须**：无论通过通知还是主动调用获取到的归因信息，都需将 @"result" 原样透传给业务后端。

如果无归因，则不添加任何监听代码。

收到回调后：
→ **核心要求**：提取 userInfo[@"result"] 或主动调用回调中的 dic，原样透传给业务后端（无论成功或失败）。

---

## Step 8：IDFA 控制（ATT 配置）

如果提示词包含“ATT”（仅用于启用 IDFA 支持，不添加任何 ATT 权限请求代码）：

- 在 Info.plist 添加（或确认存在）：

```
<key>NSUserTrackingUsageDescription</key>
<string>我们使用您的广告标识符来提供个性化体验。</string>
```

- 创建或编辑 PrivacyInfo.xcprivacy 文件（必须；如果文件不存在，使用命令行在项目根目录或 Supporting Files 文件夹创建）：

  ```bash
  touch PrivacyInfo.xcprivacy
  ```

  然后写入完整内容：

  ```bash
  cat > PrivacyInfo.xcprivacy << EOF
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
      <key>NSPrivacyTracking</key>
      <true/>
      <key>NSPrivacyTrackingDomains</key>
      <array/>
  </dict>
  </plist>
  EOF
  ```

  如果文件已存在，仅需确保包含 `<key>NSPrivacyTracking</key><true/>` 和 `<key>NSPrivacyTrackingDomains</key><array/>`。

- 在初始化 config 中设置：Swift - config.idfaEnable = true；OC - config.idfaEnable = YES。

否则（无 ATT）：Swift - config.idfaEnable = false；OC - config.idfaEnable = NO；不配置以上隐私文件和描述。

规则：
- **严禁添加任何 ATT 权限请求代码**（如 import AppTrackingTransparency 或 ATTrackingManager.requestTrackingAuthorization）。
- ATT 配置仅限于启用 IDFA 收集（idfaEnable + 隐私声明文件），权限弹出由 SDK 或业务方自行处理。

---

## Step 9：验证

验证方式：
- 启动 App，检查 console 是否有统计日志  
- 如果接入数数：后台 Debug 模式是否有事件  
- 如果接入 AppsFlyer/Adjust：是否有启动事件  

---

## Step 10：集成总结

总结说明：
- 设备唯一标识 ZT 使用方式
- 用户与会员状态使用方式
- 验证方法

---

## 约束规则（AI 必须遵守）

1. 必须先处理隐私协议再初始化  
2. AppsFlyer 与 Adjust 不能同时接入（基于提示词二选一，或不选）  
3. userIdentifier 不允许缓存  
4. 生命周期方法不能加条件判断  
5. 初始化只能在同意隐私后执行  
6. 所有代码添加前，必须在类中导入相应头文件  
7. Pod install 前检查并添加私有源索引  
8. 基于提示词动态填充参数和配置；在代码中用占位符如“你的appid”显示参数，让用户替换  
9. 判断项目语言（Swift/OC），生成相应代码；Swift 使用直接调用，无桥接  
10. 如果是 SwiftUI 项目，使用 scenePhase 处理生命周期，并在 App struct 或 View 中处理初始化和监听  
11. ATT 配置仅限于 idfaEnable = true + Info.plist 描述 + PrivacyInfo.xcprivacy 文件声明；**禁止生成任何 ATT 权限请求代码**  
12. 归因监听必须使用正确的通知名，并在回调中提取并原样透传 userInfo[@"result"] 给业务后端  
13. **生命周期判断**：AI 必须检查项目文件结构：
   - 若存在 SceneDelegate → 隐私控制、初始化、前后台代码全部添加到 SceneDelegate 的相应方法（sceneDidBecomeActive / sceneWillResignActive）
   - 若为 SwiftUI 项目 → 使用 scenePhase 处理前后台，初始化放在 init() 或 onAppear
   - 否则 → 使用 AppDelegate
14.最后总结说明不要生成文档，直接输出文字说明。

---

## AI 调用示例

```
访问文档，帮我集成统计SDK、ATT、AppsFlyer、数数。
```

AI 响应：先判断项目结构（AppDelegate / SceneDelegate / SwiftUI），生成对应 Podfile 修改、代码片段（参数用占位符）、Info.plist 更新、PrivacyInfo.xcprivacy 创建命令、执行 pod install。