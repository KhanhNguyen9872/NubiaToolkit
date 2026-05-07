# Super Resolution / Superior Pic Quality analysis

## Mục tiêu
- Map hook hiện có của `NubiaToolkit` vào smali thật của `game_assist` và `game_launcher`
- Xác định logic check support theo package nằm ở đâu
- Đánh giá khả năng làm `no-root`, `no-hook`
- Xác định chỗ nào có thể thử bằng ADB

---

## 1. Hook của toolkit đang đụng vào đâu

File hook:
- `app/src/main/java/com/khanhnguyen9872/nubiatoolkit/hooks/SuperResolutionHook.java`

Hook target:
- `cn.nubia.gameassist`
- `cn.nubia.gamelauncher`

Hook map sang smali thật:
- `PluginUtils` → `game_assist/classes_smali/cn/nubia/gameassist/plugin/PluginUtils.smali`
- `SuperResolutionTypeDataManager` → `game_assist/classes_smali/cn/nubia/plugin/superresolution/SuperResolutionTypeDataManager.smali`
- `ZteFeature` → `game_assist/classes_smali/com/zte/gameassist/config/ZteFeature.smali`
- `PluginConfig` → `game_assist/classes_smali/cn/nubia/gameassist/plugin/config/PluginConfig.smali`
- `Utils.isSmallWindowOpen` → `game_assist/classes_smali/cn/nubia/gameassist/utils/Utils.smali`
- `SuperResolutionHelper` → `game_launcher/classes_smali/cn/nubia/gamelauncher/gamecontrolpanel/superresolution/SuperResolutionHelper.smali`
- `ControlPanelFeatureHelper` → `game_launcher/classes_smali/cn/nubia/gamelauncher/gamecontrolpanel/utils/ControlPanelFeatureHelper.smali`

---

## 2. `game_launcher` support/apply path thực sự như thế nào

File chính:
- `game_launcher/classes_smali/cn/nubia/gamelauncher/gamecontrolpanel/superresolution/SuperResolutionHelper.smali`
- `game_launcher/classes_smali/cn/nubia/gamelauncher/service/GameFeatureService.smali`
- `game_launcher/classes_smali/cn/nubia/gamelauncher/gamecontrolpanel/superresolution/ReflectSystemPropertiesUtil.smali`

### 2.1 White list hardcode có tồn tại, nhưng không phải gate chính đang được dùng
Đoạn init:
- `SuperResolutionHelper.smali:52-115`

Các field liên quan:
- `SUPPORT_SUPER_RESOLUTION_WHITE_LIST`
- `PACKAGE_MAPPING_MAP`
- `CURRENT_APP_SUPPORT_GEAR_ARRAY`

Hardcode thấy được:
- `com.miHoYo.Yuanshen` được add vào `SUPPORT_SUPER_RESOLUTION_WHITE_LIST`

Mapping thêm:
- `com.miHoYo.Yuanshen` → `yuanshen`
- `com.miHoYo.ys.bilibili` → `yuanshen`
- `com.miHoYo.ys.mi` → `yuanshen`

Method whitelist check:
- `supportSuperResolutionByPkgName(String pkg)` tại `SuperResolutionHelper.smali:2005-2047`
- logic chỉ là `SUPPORT_SUPER_RESOLUTION_WHITE_LIST.contains(pkg)`

Phát hiện quan trọng:
- trong `game_launcher` smali hiện đọc được, **không thấy caller thực tế** của `supportSuperResolutionByPkgName()`
- nghĩa là whitelist helper này có thể là logic cũ / helper phụ / dead path
- **không nên coi nó là gate cuối cùng duy nhất**

### 2.2 Gate chính đang dùng thật: `getSupportResolutionGear(pkg)`
Method:
- `SuperResolutionHelper.smali:1282-1617`

Đây mới là đường quyết định package có gear nào để mở SR.

Nó có 2 mode:

#### Mode A — detach mode bật
Cờ:
- `ZTE_FEATURE_DISPLAY_MAGIC_DETACH_ENABLE`
- launcher: `ControlPanelFeatureHelper.smali:476-489`
- assist: `ZteFeature.smali:514-527`

Nếu bật:
- đọc `/data/gamemagic/config/magic_config.xml`
- parse `app_map_config > config resolution_config > item package_name`
- match exact `package_name`
- lấy `resolution_config` tương ứng

Refs:
- parse XML: `SuperResolutionHelper.smali:764-1210`
- select gear by pkg: `SuperResolutionHelper.smali:1307-1469`

#### Mode B — detach mode tắt
Nếu tắt:
- map package qua `PACKAGE_MAPPING_MAP`
- tra token trong `ControlPanelFeatureHelper.getZteFeatureMagicResolutions()`
- nếu không có token match → fallback `default`
- nếu support string rỗng → fallback `DEFAULT_SUPPORT_ITEM = {"1116", "1440"}`

Refs:
- `SuperResolutionHelper.smali:1475-1617`
- `ControlPanelFeatureHelper.smali:521-534`

Kết luận cập nhật:
- `game_launcher` **không chỉ** dựa vào hardcoded whitelist
- gate support thực tế thiên về:
  - detach XML path, hoặc
  - feature string `ZTE_FEATURE_MAGIC_RESOLUTIONS`, hoặc
  - default gear fallback

### 2.3 Open/close SR trong launcher không apply effect trực tiếp; nó ghi state rồi yêu cầu restart app
`GameFeatureService` path:
- `onStartCommand()` → lấy pkg → gọi `getSupportResolutionGear(pkg)`
- null gear → abort
- nếu nhiều hơn 1 gear → show settings dialog
- nếu 1 gear → open trực tiếp
- sau open/close → show restart app dialog

Refs:
- `GameFeatureService.smali:370-427`
- `GameFeatureService.smali:445-607`

#### open path
- `SuperResolutionHelper.openSuperResolution(Context, pkg, int)`
- build key `persist.maso.<pkg/suffix>`
- gọi reflection vào `android.os.SystemProperties.set(key, value)`

Refs:
- `SuperResolutionHelper.smali:1620-1657`
- `constructPropertiesKeyByPkgName()` → `SuperResolutionHelper.smali:369-462`
- `ReflectSystemPropertiesUtil.set()` → `ReflectSystemPropertiesUtil.smali:227-305`

#### close path
- `SuperResolutionHelper.closeSuperResolution(Context, pkg)`
- set sysprop key đó về `0`

Refs:
- `SuperResolutionHelper.smali:339-366`
- `ReflectSystemPropertiesUtil.set()` → `ReflectSystemPropertiesUtil.smali:227-305`

#### fallback/last-state path
- lưu `super_resolution_switch_key` trong `Settings.Global`
- đọc/ghi ở:
  - `getLastSuperResolutionSwitchStats()` → `SuperResolutionHelper.smali:498-648`
  - `saveLastSuperResolutionSwitchStats()` → `SuperResolutionHelper.smali:1673-1932`

#### path phụ ngoài `GameFeatureService`
Ngoài service path, `GameControlDialog` cũng có nhánh tự ghi lại launcher-side state:
- nếu resolve được key pkg → gọi `ReflectSystemPropertiesUtil.set(...)`
- sau đó đọc/ghép lại `Settings.Global["super_resolution_switch_key"]`

Ref:
- `GameControlDialog.smali:1677-1749`

=> nghĩa là launcher-side state write không chỉ nằm ở 1 service method; UI dialog path cũng có thể chạm trực tiếp vào prop/global cache.

#### hidden implication
Launcher Java code **không thấy call native/render apply trực tiếp**.
Nó làm chủ yếu các việc:
1. chọn gear
2. ghi sysprop / global state
3. yêu cầu restart app

=> effect thật nhiều khả năng được apply ở tầng dưới (vendor/native/service khác) sau khi app/game restart.

### 2.4 Hai prop family liên quan trong launcher
Trong `ReflectSystemPropertiesUtil`:
- global prop default: `persist.magic.super.resolution`
- per-pkg prefix: `persist.maso.`
- class phản xạ thêm `com.redmagic.os.RedMagicAppManager` / `Trigger`

Nhưng có 2 nhánh reflection khác nhau:
1. `get()` / `set()`
   - load `android.os.SystemProperties`
   - default key rỗng → `persist.magic.super.resolution`
2. `getSystemProperties()` / `setSystemProperties()`
   - load `com.redmagic.os.RedMagicAppManager`
   - gọi method `getSystemProperties` / `setSystemProperties`
   - cũng trỏ vào `persist.magic.super.resolution`

Refs:
- `ReflectSystemPropertiesUtil.smali:7-25`
- `ReflectSystemPropertiesUtil.smali:38-135`
- `ReflectSystemPropertiesUtil.smali:137-225`
- `ReflectSystemPropertiesUtil.smali:227-305`
- `ReflectSystemPropertiesUtil.smali:380-456`

Kết luận bổ sung:
- repo smali cho thấy launcher có sẵn cả đường `SystemProperties` thuần Java lẫn đường bridge qua `RedMagicAppManager`
- nhưng grep caller trong repo hiện tại chỉ cho thấy SR flow đang gọi rõ ràng vào nhánh `set()`/`get()`; chưa thấy caller trực tiếp của `setSystemProperties()` trong flow SR Java đã trace
- điều này lại tăng khả năng phần apply thật nằm ngoài flow Java hiện đọc được, hoặc runtime build khác repo đang dùng nhánh khác

---

## 3. `game_assist` support/runtime path thực sự như thế nào

File chính:
- `game_assist/classes_smali/cn/nubia/gameassist/plugin/PluginUtils.smali`
- `game_assist/classes_smali/cn/nubia/plugin/superresolution/SuperResolutionViewController.smali`
- `game_assist/classes_smali/cn/nubia/plugin/superresolution/SuperResolutionTypeDataManager.smali`
- `game_assist/classes_smali/cn/nubia/plugin/superresolution/SuperResolutionSettingWindowManager.smali`

### 3.1 XML/JSON config path vẫn tồn tại trong code
`PluginUtils` path:
- key: `magic_config_resolution`
- file: `data/gamemagic/config/magic_config.xml`

Refs:
- field/path: `PluginUtils.smali:19-21`
- parse current pkg: `PluginUtils.smali:141-259`
- parse XML `package_name`: `PluginUtils.smali:262-488`
- read file: `PluginUtils.smali:676-757`

`PluginUtils$ResolutionConfig.saveConfigToLocal()`:
- không lưu file riêng
- nó format JSON map rồi ghi vào prefs key `magic_config_resolution`

Refs:
- `PluginUtils$ResolutionConfig.smali:121-218`
- `JsonUtil.smali:44-189`, `JsonUtil.smali:191-318`

Format local map kiểu:
```json
{"json_list":{"pkg.a":"1116,1440","pkg.b":"1116"}}
```

### 3.2 Nhưng gate runtime mạnh hơn XML/prefs là `getGfrcCapByPkg(pkg)`
`PluginUtils.getGfrcCapByPkg(pkg)`:
- reflection sang `com.zte.performance.mindsync.MindSyncManager$Trigger.getGfrcCapByPkg`

Refs:
- `PluginUtils.smali:492-590`

`supportResolution(pkg)` chỉ là:
- `cap != 0`

Ref:
- `PluginUtils.smali:759-777`

=> XML/prefs có thể làm package được recognize, nhưng **cap runtime mới là gate lõi**.

### 3.3 `SuperResolutionTypeDataManager` chỉ giữ per-pkg state, không tự cấp quyền support
Method:
- `getItem(pkg, type)` → nếu không có item trả:
  - `imageQuality` → `origin`
  - `frameRate` → `frameRate_origin`
- `putItem(pkg, type, value)` → update list rồi save lại prefs

Refs:
- `getItem()` → `SuperResolutionTypeDataManager.smali:189-300`
- `putItem()` → `SuperResolutionTypeDataManager.smali:367-481`
- `saveList()` → `SuperResolutionTypeDataManager.smali:484-529`

Kết luận:
- `plugin_super_resolution_typeItem_Data` chỉ là state lựa chọn / effective mode snapshot
- **không phải source support gốc**

### 3.4 `game_gfrc_mode` là runtime state channel quan trọng nhất
Field/key:
- `SETTINGS_SUPPORT_RESOLUTION_ENABLE_PKGS = "game_gfrc_mode"`

Ref:
- `SuperResolutionViewController.smali:28`

Format chuỗi:
```text
pkg+XYZ,pkg+XYZ,...
```

Trong đó:
- digit 0 (`X`) = image quality
- digit 1 (`Y`) = frame rate
- digit 2 (`Z`) = master switch

Parse path:
- `getPkgSwitchMap()` → đọc `Settings.Global["game_gfrc_mode"]`
- `parseSettingsValue()` → split `,` rồi `+`
- `handleSwitchValue(pkg, idx, digit)` → dispatch từng digit

Refs:
- `getPkgSwitchMap()` → `SuperResolutionViewController.smali:357-475`
- `parseSettingsValue()` → `SuperResolutionViewController.smali:832-952`
- `handleSwitchValue()` → `SuperResolutionViewController.smali:555-629`

### 3.5 Ý nghĩa chính xác của từng digit trong `game_gfrc_mode`
#### digit 0 — image quality
- `0` → `origin`
- `1` → `high`
- `2` → `super`

Refs:
- `getImageQuality()` → `SuperResolutionViewController.smali:247-280`
- `getImageQualitySwitch()` → `SuperResolutionViewController.smali:283-317`

#### digit 1 — frame rate
- `0` → `frameRate_origin`
- `1` → `frameRate_super`

Refs:
- `getFrameRate()` → `SuperResolutionViewController.smali:201-221`
- `getFrameRateSwitch()` → `SuperResolutionViewController.smali:224-245`

#### digit 2 — master switch
- thường: `1` = open, `0` = close
- PUBG list special case:
  - `6` = open
  - `5` = close
- `8` = thermal-forced close + show toast nhiệt độ cao

Refs:
- constants: `PUBG_LIST_OPEN`, `PUBG_LIST_CLOSE` → `SuperResolutionViewController.smali:24-28`
- PUBG list init → `SuperResolutionViewController.smali:109-124`
- `isSwitchOpen()` → `SuperResolutionViewController.smali:652-702`
- thermal toast path → `SuperResolutionViewController.smali:570-603`

### 3.6 Ý nghĩa của runtime string thật trên máy
Device dump:
```text
settings get global game_gfrc_mode
com.ludashi.benchmark+210,com.antutu.ABenchMark+210,com.miHoYo.GenshinImpact.vn+211,emuready.gamehub.lite+210,org.citron.citron_emu+210,com.garena.game.kgvn+210,catch_.me_.if_.you_.can_+210,com.xiaoji.egggame+210,com.pqvxw.wwiw+210,com.SyGame.Capitalism+210
```

Giải mã:
- `com.miHoYo.GenshinImpact.vn+211`
  - imageQuality = `super`
  - frameRate = `frameRate_super`
  - master switch = on
- các pkg `+210`
  - imageQuality = `super`
  - frameRate = `frameRate_super`
  - master switch = off

Kết luận rất quan trọng:
- tại thời điểm dump này, với `com.miHoYo.GenshinImpact.vn`, **assist-side runtime state đang full ON**
- điều này mạnh hơn snapshot prefs cũ có `origin`
- nếu prefs copy vẫn còn `origin`, nhiều khả năng đó là snapshot lệch thời điểm / chưa flush / copy nhầm path/time

### 3.7 Cap integer có cấu trúc 2 phần và có thể force reset state
`parseSupportFunction(cap)`:
- `cap / 10` = interpolation / frame-rate support
- `cap % 10` = super-resolution support

Refs:
- `SuperResolutionViewController.smali:954-1076`
- `SuperResolutionSettingWindowManager.smali:908+`

Force-reset behavior:
- nếu `cap == 0`
  - force `frameRate = frameRate_origin`
  - force `imageQuality = origin`
  - mark first-open
  - disable switch
- nếu `cap / 10 == 0`
  - force `imageQuality = origin`
- nếu `cap % 10 == 0`
  - force `frameRate = frameRate_origin`

=> đây là hidden gate rất mạnh. Package có row trong prefs vẫn có thể bị ép về default khi game start nếu cap không đủ.

### 3.8 Performance mode là gate phụ nhưng đủ để disable switch
`checkGameMode(pkg, mode)`:
- nếu economize/balance mode
  - `updateEnableSwitchPkg(pkg, false)`
  - `putFirstOpenPkg(pkg)`

Refs:
- `SuperResolutionViewController.smali:179-198`

Ngoài ra observer của `game_gfrc_mode` chỉ reparse khi performance mode không phải economize/balance.

### 3.9 `onGameStart()` flow đầy đủ
`SuperResolutionViewController.onGameStart()`:
1. load `plugin_super_resolution_typeItem_Data`
2. lấy current pkg
3. gọi `PluginUtils.getGfrcCapByPkg(pkg)`
4. `parseSupportFunction(cap)`
5. đọc current performance mode và `checkGameMode(pkg, mode)`
6. đọc `Settings.Global["game_gfrc_mode"]`
7. `parseDataOnGameStart(...)`
8. nếu prefs item data null → clear `game_gfrc_mode`
9. register observer

Refs:
- `SuperResolutionViewController.smali:1579-1681`

=> `game_gfrc_mode` là state runtime thật, nhưng vẫn chịu ảnh hưởng của cap + game mode mỗi lần game start.

---

## 4. SharedPreferences/runtime files thật trên thiết bị
### 4.1 Files thấy được
Từ thiết bị:
- `gameassist/shared_prefs`
  - `data.xml`
  - `nubia_game_plugin.xml`
  - file phụ khác
- `gamelauncher/shared_prefs`
  - `data.xml`
  - prefs phụ khác

### 4.2 Nội dung quan trọng từ `data.xml` snapshot đã copy ra project
Keys thấy được:
- `plugin_super_resolution_typeItem_Data`
- `plugin_first_switch_pkg_super_resolution`
- `custom_defined_fps_com.miHoYo.GenshinImpact.vn = 120`
- `mode_plugin = {"json_list":{"com.miHoYo.GenshinImpact.vn":1,...}}`

Ý nghĩa:
- package `com.miHoYo.GenshinImpact.vn` **đã được subsystem recognize**
- đã có per-pkg SR row
- đã có workflow/plugin state

Nhưng lưu ý:
- snapshot prefs cũ từng cho thấy row còn `origin` / `frameRate_origin`
- trong khi `game_gfrc_mode` runtime mới nhất lại là `+211`
- do đó, với debugging hiện tại, **nên tin `game_gfrc_mode` hơn snapshot prefs rời**

### 4.3 Điều chưa thấy trên thiết bị
Không thấy:
- `magic_config_resolution` trong snapshot `data.xml` đã copy
- `/data/gamemagic/config/magic_config.xml` trên máy (`/data/gamemagic` không tồn tại lúc kiểm tra)

=> XML detach path có trong code, nhưng **không có dấu vết runtime trên máy tại thời điểm check**.

---

## 5. Feature flags và gate phụ khác
### 5.1 Feature flags hệ thống
File:
- `game_assist/classes_smali/com/zte/gameassist/config/ZteFeature.smali`

Methods:
- `isSupportSuperResolution()` → đọc `ZTE_FEATURE_GFRC`
- `isSupportSuperResolutionOld()` → đọc `ZTE_FEATURE_MAGIC_SUPER_RESOLUTION`
- `isSupportSuperResolutionSettings()` → đọc `ZTE_FEATURE_MAGIC_RESOLUTIONS_SETTINGS`
- `isSuperResolutionDetachEnable()` → đọc `ZTE_FEATURE_DISPLAY_MAGIC_DETACH_ENABLE`

Refs:
- `ZteFeature.smali:514-527`, `1076-1118`

Trong launcher:
- `ControlPanelFeatureHelper.getZteFeatureMagicSuperResolution()` → `ControlPanelFeatureHelper.smali:536-548`
- `ControlPanelFeatureHelper.getZteFeatureDisplayMagicDetachEnable()` → `ControlPanelFeatureHelper.smali:476-489`

### 5.2 Small window gate
File:
- `game_assist/classes_smali/cn/nubia/gameassist/utils/Utils.smali:2555-2577`

Method:
- `isSmallWindowOpen(Context)`
- đọc `Settings.Secure["hasWindowReply"]`
- nếu `"1"` → coi như small window đang mở

Hook của toolkit force `false` để bỏ chặn này.

---

## 6. Vì sao hook hiện tại hiệu quả
Hook hiện tại bypass đồng thời nhiều lớp:

1. **Feature/system gates**
   - `ZTE_FEATURE_GFRC`
   - `ZTE_FEATURE_MAGIC_SUPER_RESOLUTION`
   - `ZTE_FEATURE_MAGIC_RESOLUTIONS_SETTINGS`
   - `ZTE_FEATURE_DISPLAY_MAGIC_DETACH_ENABLE`

2. **Package/config gates**
   - launcher gear path (`getSupportResolutionGear`)
   - assist XML/JSON path (`magic_config_resolution`)
   - possible cap gate from `MindSyncManager`

3. **Runtime state gates**
   - `SuperResolutionTypeDataManager.getItem()` defaults
   - `game_gfrc_mode`
   - prefs `plugin_enable_pkg_super_resolution`
   - prefs `plugin_first_switch_pkg_super_resolution`
   - prefs `plugin_super_resolution_typeItem_Data`

4. **UI / environment gates**
   - small-window gate
   - performance mode gate

=> hook không trực tiếp tạo effect render; nó làm cho toàn bộ app stack tin rằng:
- máy hỗ trợ
- package hợp lệ
- cap hợp lệ
- state bật
- UI được phép hiện
- panel/settings không tự reset về off/default

---

## 7. Kết quả runtime thật trên thiết bị hiện tại
### 7.1 Package/runtime info
- `cn.nubia.gameassist`
  - system app gốc tại `/system/app/GameAssist`
- `cn.nubia.gamelauncher`
  - **updated system app** chạy từ `/data/app/.../base.apk`
  - vẫn còn hidden original ở `/system/priv-app/GameSpace`

Ý nghĩa:
- smali local trong repo có thể **không khớp 100%** với APK runtime hiện tại trên máy
- mọi kết luận launcher-side cần luôn nhớ khả năng **version drift**

### 7.2 State quan sát được
Đã bật cho `com.miHoYo.GenshinImpact.vn`.

Kết quả:
```bash
settings get global game_gfrc_mode
# có com.miHoYo.GenshinImpact.vn+211

settings get global super_resolution_switch_key
# null

getprop | grep -i "maso\|magic.super.resolution"
# không ra gì
```

### 7.3 Ý nghĩa của split này
- `game_assist` side: **ON thật** (`+211`)
- `game_launcher` Java apply-state theo smali local:
  - **không thấy** `super_resolution_switch_key`
  - **không thấy** `persist.maso.*`
  - **không thấy** `persist.magic.super.resolution`

Trong khi đó repo launcher lại có ít nhất 2 chỗ có thể ghi state:
- `GameFeatureService` → `SuperResolutionHelper.open/close/saveLast...`
- `GameControlDialog` → `ReflectSystemPropertiesUtil.set(...)` + xử lý `super_resolution_switch_key`

=> nếu device runtime vẫn không lộ bất kỳ dấu vết nào ở prop/global layer, thì khả năng lệch repo-vs-runtime càng mạnh hơn: không phải chỉ miss 1 caller, mà gần như cả launcher-side state-write model trong repo đều không khớp dấu vết runtime hiện thấy.

=> có 2 khả năng lớn:
1. **runtime APK drift**: launcher runtime trên máy không còn dùng y nguyên apply path như smali local trong repo
2. **assist state on ≠ engine/render apply on**: state/UI đã bật nhưng tầng vendor/native apply effect thật chưa được trigger

### 7.4 Kết luận mạnh nhất hiện tại
Với evidence mới nhất:
- case này **không còn** là “package unsupported nên luôn origin”
- assist-side runtime state cho Genshin VN đang full on
- nếu visual effect thực tế vẫn không thấy, blocker load-bearing nhiều khả năng nằm ở:
  - launcher runtime khác build
  - vendor/native service consumer của prop/state
  - engine/render apply layer thấp hơn Java smali đang đọc

---

## 8. Khả năng làm `no-root`, `no-hook`
### Kết luận ngắn cập nhật
Nếu yêu cầu là:
- app hệ thống gốc
- không root
- không hook/inject
- không cùng OEM/platform signature

thì thay đổi full logic/apply path của app gốc vẫn là **rất khó / gần như không khả thi đầy đủ**.

### Vì sao
- app là system / updated system app
- launcher runtime có dấu hiệu lệch với smali local
- assist-side state bật chưa đủ chứng minh render layer bật
- phần apply thật có thể nằm ở vendor/native/service ngoài APK Java layer

### Các hướng thực tế
1. **Patch APK/smali cục bộ**
   - khả thi về kỹ thuật
   - nhưng không thay app hệ thống gốc nếu khác sign

2. **Virtual/container**
   - có thể fake UI/state hoặc chạy bản patch riêng
   - không thật sự thay logic app hệ thống gốc

3. **ADB/Shizuku chỉnh state**
   - hữu ích để quan sát/ép state runtime kiểu `game_gfrc_mode`
   - nhưng chưa chứng minh được apply effect thật

4. **Root/systemless/hook**
   - vẫn là đường chắc nhất để bypass nhiều lớp cùng lúc

---

## 9. Lệnh ADB/Termux đã thử và ý nghĩa
### 9.1 Đã thử
```bash
getprop | grep -i "maso\|magic.super.resolution"
settings get global super_resolution_switch_key
settings get global game_gfrc_mode
ls /data/gamemagic
ls /data/user_de/0/cn.nubia.gameassist/shared_prefs
ls /data/user_de/0/cn.nubia.gamelauncher/shared_prefs
dumpsys package cn.nubia.gameassist
dumpsys package cn.nubia.gamelauncher
```

### 9.2 Kết quả chính
- `/data/gamemagic` không tồn tại lúc kiểm tra
- `gameassist` / `gamelauncher` đều có shared_prefs
- `game_gfrc_mode` có entry `+211` cho Genshin VN
- `super_resolution_switch_key` vẫn `null`
- `persist.maso.*` / `persist.magic.super.resolution` không thấy

---

## 10. Phán đoán hiện tại
- `game_assist` có hidden state machine riêng qua `game_gfrc_mode`
- `MindSyncManager.getGfrcCapByPkg(pkg)` vẫn là gate cap cốt lõi
- `game_launcher` support path thực tế thiên về gear/config path, không đơn thuần whitelist helper
- launcher apply path theo smali local = ghi prop/state + restart app
- runtime device evidence lại **không thấy prop/state launcher-side**, dù assist-side đã on
- vì vậy, bug/blocker hiện tại nhiều khả năng đã dịch từ “support/state layer” sang “apply/vendor/runtime layer”

---

## 11. Bước đào tiếp hợp lý
1. Kéo APK runtime thật của `cn.nubia.gamelauncher` và `cn.nubia.gameassist` từ máy ra để diff với smali local trong repo
2. Bật lại SR rồi dump logcat rộng theo các keyword:
   - `gfrc`
   - `superresolution`
   - `super_resolution`
   - `mindsync`
   - `redmagic`
   - `magic`
3. Liệt kê process/service vendor liên quan:
   - `ps -A | grep -i "redmagic\|mindsync\|game\|magic"`
   - `service list | grep -i "redmagic\|game\|magic"`
4. So snapshot đồng thời của:
   - `game_gfrc_mode`
   - `super_resolution_switch_key`
   - `getprop`
   - `data.xml` / `nubia_game_plugin.xml`
5. Nếu muốn kiểm tra giả thuyết launcher drift nhanh hơn, ưu tiên grep/diff 3 điểm này trên APK runtime:
   - caller của `ReflectSystemPropertiesUtil.set(...)`
   - caller của `saveLastSuperResolutionSwitchStats(...)`
   - caller của `setSystemProperties(...)` hoặc wrapper MindSync/RedMagic tương đương

Nếu APK runtime không còn các caller này, hoặc chuyển hết sang bridge vendor khác, thì gần như xác nhận repo local đã lệch khỏi launcher build đang chạy trên máy.

---

## Tóm tắt ngắn cập nhật
- `game_launcher` không chỉ có whitelist hardcoded; gate dùng thật là `getSupportResolutionGear(pkg)`
- `game_assist` có state runtime rất quan trọng ở `game_gfrc_mode`
- `com.miHoYo.GenshinImpact.vn+211` = assist-side full ON
- snapshot prefs cũ có thể lệch thời điểm so với runtime state thật
- launcher Java apply path theo smali local là ghi `persist.maso.*` + restart app, nhưng device runtime hiện chưa cho thấy dấu vết đó
- blocker hiện tại nhiều khả năng nằm ở **runtime APK drift hoặc vendor/native apply layer**, không còn chỉ ở prefs/package support nữa
