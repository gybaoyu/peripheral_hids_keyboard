# NCS BLE HID 键盘例程（`peripheral_hids_keyboard`）新手详解

这份工程是 Nordic NCS 里的一个**标准 BLE 外设（Peripheral）HID 键盘**示例。  
它做的事很直观：开发板通过蓝牙连接到电脑/手机后，按板子按键会“打字”（默认循环发送 `hello\n`），并支持 `Shift`、`Caps Lock`、配对安全（SMP/MITM），还可选 NFC OOB 配对。

---

## 1. 你现在这个工程在做什么

核心功能分成 4 块：

1. **BLE 外设广播与连接**：设备名 `Nordic_HIDS_keyboard`，支持最多 2 个连接/配对。  
2. **HID over GATT 键盘服务**：暴露 HID Service，发送键盘输入报告（Input Report），接收主机输出报告（Output Report，如 Caps Lock）。  
3. **按键/LED 交互**：按钮模拟按键输入；LED 显示广播状态、连接状态、Caps Lock 状态。  
4. **（可选）NFC OOB 配对**：默认开启，手机读 NFC 后触发广播并提供 LESC OOB 数据。

---

## 2. 工程结构（你最该先看哪些文件）

```text
peripheral_hids_keyboard/
├─ src/
│  ├─ main.c          # 主逻辑：BLE/HID/按键/LED/配对
│  ├─ app_nfc.c       # NFC OOB NDEF 组包和 NFC 事件
│  └─ app_nfc.h
├─ prj.conf           # 主 Kconfig 配置
├─ Kconfig            # 本示例自定义配置项（如 NFC_OOB_PAIRING）
├─ CMakeLists.txt     # 编译入口，收集 src/*.c
├─ sample.yaml        # 支持板卡和 CI 测试配置
├─ boards/            # 各板子的差异配置（栈大小、TF-M、overlay）
└─ sysbuild/ipc_radio/prj.conf  # 双核/IPC Radio 相关配置（某些 SoC 用）
```

---

## 3. 先理解 `prj.conf`（新手最关键）

`prj.conf` 决定“这个应用能不能像键盘一样稳定工作”。

### 3.1 BLE 基本能力

- `CONFIG_BT=y`：打开蓝牙栈  
- `CONFIG_BT_PERIPHERAL=y`：作为外设  
- `CONFIG_BT_DEVICE_NAME="Nordic_HIDS_keyboard"`：设备名  
- `CONFIG_BT_MAX_CONN=2` / `CONFIG_BT_MAX_PAIRED=2`：最多 2 个连接/配对

### 3.2 安全与兼容

- `CONFIG_BT_SMP=y`：启用配对安全  
- `CONFIG_BT_HIDS_DEFAULT_PERM_RW_ENCRYPT=y`：HID 读写要求加密（很多主机更兼容）  
- `CONFIG_BT_GATT_AUTO_SEC_REQ=n`、`CONFIG_BT_AUTO_PHY_UPDATE=n`：关闭自动安全请求/PHY 更新，规避某些主机控制器 LL 过程冲突导致断开（注释里写了原因）

### 3.3 HID / BAS / DIS

- `CONFIG_BT_HIDS=y`：HID 服务  
- `CONFIG_BT_BAS=y`：电池服务（示例里做了电量模拟）  
- `CONFIG_BT_DIS=y`：设备信息服务（厂商、PID/VID 等）

### 3.4 Bond 持久化

- `CONFIG_BT_SETTINGS=y` + Flash 相关配置：把配对信息保存到 Flash，重启后可记住绑定设备。

---

## 4. `Kconfig` 的重点：NFC OOB 默认开启

`Kconfig` 里定义了：

- `config NFC_OOB_PAIRING`（默认 `y`，且芯片需有 NFCT 外设）  
- 选择了 `NFC_T2T_NRFXLIB`、`NFC_NDEF_*`、`NFC_NDEF_LE_OOB_REC` 等依赖

含义：默认不是上电就广播，而是**先等手机触发 NFC 场**，再开始广播并走 LESC OOB 数据流程。  
如果你是纯 BLE 新手，想先简化流程，可把它关掉（见后文“快速上手建议”）。

---

## 5. `src/main.c` 主流程（按执行顺序）

## 5.1 `main()` 启动序列

1. 打印启动日志  
2. `configure_gpio()`：初始化开发板按钮/LED  
3. 注册配对回调：  
   - `bt_conn_auth_cb_register()`（显示/确认 passkey、取消、OOB 请求）  
   - `bt_conn_auth_info_cb_register()`（配对完成/失败通知）  
4. `hid_init()`：初始化 HID 服务、Report Map、回调  
5. `bt_enable(NULL)`：启动蓝牙  
6. `settings_load()`：加载已保存绑定信息  
7. 根据 `CONFIG_NFC_OOB_PAIRING`：  
   - 开：`app_nfc_init()`，等待 NFC 触发  
   - 关：直接 `advertising_start()`  
8. 主循环：广告灯闪烁 + 周期性更新 BAS 电量

---

## 5.2 广播与连接管理

### 广播数据

`ad[]`（广播包）里放了：

- Appearance（键盘外观）  
- Flags（LE General Discoverable + BR/EDR not supported）  
- 16-bit UUID：HIDS + BAS

`sd[]`（扫描响应）里放完整设备名。

### 连接回调

- `connected()`：  
  - 通知 HIDS 库已连接 `bt_hids_connected()`  
  - 记录连接槽 `conn_mode[i]`（还记录 boot/report 模式）  
  - 无 NFC 模式下可继续保持多连接广播
- `disconnected()`：  
  - 通知 HIDS 库断开 `bt_hids_disconnected()`  
  - 清槽位、更新连接 LED  
  - NFC 模式：若在广播则停止（示例行为）  
  - 非 NFC 模式：自动重启广播

---

## 5.3 HID 核心：Report Map + 收发

`hid_init()` 里定义了键盘 report map（手写字节数组）：

- **Input Report**：  
  - 1 字节 modifier（Ctrl/Shift/Alt 等）  
  - 1 字节保留  
  - 6 字节普通键值（最多 6 键同时按）
- **Output Report**：  
  - LED 位（Num Lock/Caps Lock 等）

发送路径：

- 维护全局状态 `hid_keyboard_state`
- `hid_buttons_press()/release()` 修改状态
- `key_report_send()` 对每个连接发送
- 单连接发送用 `key_report_con_send()`，根据协议模式选：
  - `bt_hids_inp_rep_send()`（Report mode）
  - `bt_hids_boot_kb_inp_rep_send()`（Boot mode）

接收路径：

- `hids_outp_rep_handler()` 与 `hids_boot_kb_outp_rep_handler()` 接收主机写入的 output report  
- `caps_lock_handler()` 解析 bit1（0x02）并点亮/熄灭 Caps Lock LED

---

## 5.4 按键逻辑（你按一下按钮后发生什么）

默认按键映射（nRF52/nRF53 常见板）：

- `Button 1`：发送文本下一个字符  
- `Button 2`：Shift  
- `Button 4`：NFC 模式下手动触发广播

> nRF54 系列在 README 中对应的是 Button0/1/3 与 LED0/1/2/3 的映射，底层通过 DK 库抽象。

处理入口是 `button_changed()`：

1. 若当前处于 MITM passkey 确认流程，优先把按钮当“确认/拒绝”处理  
2. 否则处理文本键和 Shift 键  
3. 若使能 NFC 且按了广告键，则尝试手动启动广播

文本发送细节：

- `hello_world_str[] = {h,e,l,l,o,Return}`（HID key code）  
- 按下发送“按下报告”，松开发送“释放报告”  
- 每次松开后指针移到下一个字符，循环往复

---

## 5.5 配对安全（SMP/MITM）

关键点：

- `auth_passkey_display()`：打印 passkey  
- `auth_passkey_confirm()`：把确认请求放入 `mitm_queue` 队列  
- `pairing_work/pairing_process()`：串行处理确认提示，避免多连接同时刷屏  
- `num_comp_reply(true/false)`：按钮确认或拒绝  
- `pairing_complete()/pairing_failed()`：结果日志

这套实现很实用：即便多个设备同时发起配对，也通过消息队列顺序处理，不会乱。

---

## 6. `src/app_nfc.c`：NFC OOB 是怎么接到 BLE 的

当 `CONFIG_NFC_OOB_PAIRING=y`：

1. `app_nfc_init()` 初始化 Type2 Tag 仿真  
2. `nfc_ndef_le_oob_msg_encode()`：
   - `bt_le_oob_get_local()` 获取本地 LESC OOB 数据  
   - 组 NDEF LE OOB Record（地址、SC 数据、设备名、外观、flag）  
3. `nfc_t2t_payload_set()` 设置 NFC payload  
4. `nfc_t2t_emulation_start()` 开始 NFC 感应  
5. `NFC_T2T_EVENT_FIELD_ON` -> `nfc_field_detected()`：点亮 NFC LED，并在有空闲连接槽时提交广播启动工作

简化理解：**手机碰一下 NFC，就把配对所需信息给到手机并触发 BLE 广播。**

---

## 7. 多板卡差异配置你怎么看

`boards/` 下是“同一应用在不同 SoC/板子上的补丁配置”：

- 多个 `nrf54*cpuapp.conf`：把 `CONFIG_MAIN_STACK_SIZE` 调到 `3584`  
- `nrf5340..._ns.conf`：TF-M profile 设为 `SMALL`（非安全域应用时常见）  
- `nrf54h20...overlay`：使能 `&nfct` 节点并指定 DMA 区域  

`sysbuild/ipc_radio/prj.conf` 则是 IPC Radio/双核架构相关（BT HCI RAW、IPC service 等）。

---

## 8. 新手调试时最常看的日志

串口里你会看到典型日志：

- `Starting Bluetooth Peripheral HIDS keyboard sample`  
- `Bluetooth initialized`  
- `Advertising successfully started`（非 NFC 模式会马上出现）  
- `Connected ...` / `Disconnected ...`  
- `Passkey for ...`、`Pairing completed/failed ...`

如果你能稳定看到这些日志，说明主流程基本通了。

---

## 9. 给 NCS 新手的“最省事上手法”

建议第一阶段先**关 NFC OOB**，专注 BLE HID 主链路：

1. 在 `prj.conf` 增加（或在 menuconfig 里关）：
   - `CONFIG_NFC_OOB_PAIRING=n`
2. 烧录后上电应直接广播  
3. Windows/macOS/手机连接设备名 `Nordic_HIDS_keyboard`  
4. 打开文本框，按开发板文本键，观察 `hello` 输入  
5. 按 Shift 键组合，确认大写  
6. 在主机开关 Caps Lock，观察开发板 Caps LED

等这条链路稳定后，再打开 NFC OOB，会更容易理解问题在哪一层。

---

## 10. 常见问题（新手高频）

### Q1：为什么连上又断开？

- 主机蓝牙控制器兼容问题常见。这个示例已通过  
  `CONFIG_BT_GATT_AUTO_SEC_REQ=n`、`CONFIG_BT_AUTO_PHY_UPDATE=n`  
  做了规避。  
- 另外检查供电、距离、是否有历史错误 bond（可清空后重配）。

### Q2：为什么按键没“打字”？

- 确认连接的是系统输入框（不是只连上 BLE app）  
- 确认 HID 通知已开启（用 BLE 调试工具时）  
- 确认按下/松开事件都发生（该示例依赖 press+release 报告）

### Q3：为什么 Caps Lock LED 不变？

- 只有主机写 Output Report（bit1）才会更新  
- 有些手机/APP 不会像 PC 一样主动发这个状态

---

## 11. 这个例程值得你学习的代码点

1. **标准 HID 键盘 report map 的最小实现**  
2. **多连接状态管理**（`conn_mode[]`）  
3. **MITM 确认排队处理**（`k_msgq + k_work`）  
4. **NFC OOB 与 BLE 安全配对联动**  
5. **板级抽象（DK buttons/LED）+ 业务逻辑分层**

---

## 12. 一句话总结

这是一个“能直接当蓝牙键盘用”的完整样例：  
**主线是 BLE HIDS 输入/输出报告，安全上带 SMP/MITM，增强上带 NFC OOB。**  
你作为 NCS 新手，先跑通“无 NFC 的 HID 主链路”，再加 NFC，会学得最快。

