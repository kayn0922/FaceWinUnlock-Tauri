# 采集设备锁定（camera lock）

## 背景

Windows 下 OpenCV 的摄像头索引来自系统枚举顺序，`options.camera` 保存的是**整数索引**。
插拔任意摄像头（尤其是笔记本内置摄像头、USB 摄像头、虚拟摄像头共存时）都会让索引漂移，
原来保存的「默认采集设备」会悄悄指向另一台设备，出现「明明选的这台，却用了那台」的问题。

## 方案

锁定的是设备的 **Media Foundation symbolic link**（`MF_DEVSOURCE_ATTRIBUTE_SOURCE_TYPE_VIDCAP_SYMBOLIC_LINK`），
形如 `\\?\usb#vid_xxxx&pid_xxxx#...#{65e8773d-8f56-11d0-a3b9-00a0c9223196}`。该标识跟随设备本身，
与枚举顺序、插拔顺序无关；同型号多台摄像头也能区分。

- 设置页「采集设备」右侧新增**锁形按钮**：点击锁定当前选中的设备，再次点击解除锁定。
- 锁定记录写入 `options` 表的 `cameraLock` 字段：`{"device_id":"...","camera_name":"..."}`。
- 打开摄像头前（Unlock 服务）、枚举设备列表后（UI）、应用启动与初始化检测（App/Init）都会
  按 `device_id` 重新解析它在**当前**枚举顺序中的索引，再用该索引去打开。

## 行为约定

| 场景 | 未锁定（旧行为） | 已锁定 |
| --- | --- | --- |
| 增减/插拔其它摄像头 | 索引漂移，可能打开到别的设备 | 仍定位到锁定设备 |
| 锁定设备未接入 | 回退到列表第一个可用设备 | 不放画面、且**不回退**到其它设备（日志告警） |
| 锁定设备无有效帧 | 自动回退到其它可用设备 | 回退被禁用，避免误用其它摄像头 |
| 设备被拔出后重新插入 | 可能换到别的设备 | 重新按 symbolic link 命中 |

旧配置（仅 `camera` 索引、无 `cameraLock`）不受影响，行为与原来完全一致；只有点击锁定后才进入锁定模式。

## 排查

- 日志：`Unlock` 服务日志中 `locked camera '...' resolved to index N (saved index M)`、
  `locked camera '...' is not connected (...)`、`fallback disabled in lock mode`。
- 若 `device_id` 为空（个别虚拟摄像头驱动不返回 symbolic link），会退化为按设备友好名匹配。
