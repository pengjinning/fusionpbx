# 代码分析

## FusionPBX 与 FreeSWITCH 交互总览

FusionPBX 与 FreeSWITCH 的交互主要分为三类：

1. **主动调用 FreeSWITCH 接口（ESL API/Command）**
2. **订阅 FreeSWITCH 事件并回推到前端（ESL Event -> WebSocket）**
3. **FreeSWITCH 拉取配置与上报 CDR（Lua XML Handler / mod_xml_cdr）**

---

## 1) FusionPBX 如何调用 FreeSWITCH 接口

### 核心封装

- 统一封装在 [resources/classes/event_socket.php](resources/classes/event_socket.php)
  - `event_socket::api($cmd)` -> 发送 `api xxx`
  - `event_socket::command($cmd)` -> 发送原生命令
  - `event_socket::async($cmd)` -> 发送 `bgapi xxx`
- `connect()` 会从配置中读取：
  - `switch.event_socket.host`
  - `switch.event_socket.port`
  - `switch.event_socket.password`
- 默认值为 `127.0.0.1:8021 / ClueCon`。

关键代码：

- [resources/classes/event_socket.php](resources/classes/event_socket.php)
- [resources/switch.php](resources/switch.php)（兼容包装函数，最终仍调用 `event_socket` 类）

### 业务调用方式

大量业务页面/服务直接调用 `event_socket::api()`，例如：

- `reloadxml`、`reloadacl`
- `sofia profile ... rescan`
- `show channels as json`
- `conference ...`
- `uuid_kill ...`

常见位置（示例）：

- [app/click_to_call/click_to_call.php](app/click_to_call/click_to_call.php)
- [app/active_conferences/resources/classes/active_conferences_service.php](app/active_conferences/resources/classes/active_conferences_service.php)
- [app/call_centers](app/call_centers)
- [app/gateways](app/gateways)

---

## 2) FreeSWITCH 事件如何通知 FusionPBX

### 总体链路

1. FusionPBX 的系统服务进程连接 FreeSWITCH ESL。
2. 发送 `event plain all` 并追加 `filter ...`（只订阅需要的事件）。
3. 循环 `read_event()` 读取事件并转成内部 `event_message`。
4. 通过内部 WebSocket 服务广播到已订阅的浏览器客户端。

### Active Calls 服务示例

- 服务入口：[app/active_calls/resources/service/active_calls.php](app/active_calls/resources/service/active_calls.php)
- 关键类：[app/active_calls/resources/classes/active_calls_service.php](app/active_calls/resources/classes/active_calls_service.php)
  - `connect_to_event_socket()`：连接并认证 ESL
  - `register_event_socket_filters()`：注册 `Event-Name/Event-Subclass` 过滤器
  - `handle_switch_event()`：接收事件后通过 `websocket_client::send(...)` 推送

### Active Conferences 服务示例

- 服务入口：[app/active_conferences/resources/service/active_conferences.php](app/active_conferences/resources/service/active_conferences.php)
- 关键类：[app/active_conferences/resources/classes/active_conferences_service.php](app/active_conferences/resources/classes/active_conferences_service.php)
  - `connect_to_switch_server()`
  - `register_event_socket_filters()`
  - `handle_switch_events()`

### WebSocket 中心路由

- 入口：[core/websockets/resources/service/websockets.php](core/websockets/resources/service/websockets.php)
- 核心：[core/websockets/resources/classes/websocket_service.php](core/websockets/resources/classes/websocket_service.php)
  - 负责服务端与浏览器端认证、订阅、消息路由和广播。

---

## 3) FreeSWITCH -> FusionPBX 的其他通道

### 3.1 配置拉取（XML Handler）

FreeSWITCH 通过 Lua XML Handler 动态获取配置，而不是仅使用静态 XML 文件。

- [app/switch/resources/conf/autoload_configs/lua.conf.xml](app/switch/resources/conf/autoload_configs/lua.conf.xml)
  - `xml-handler-script = app.lua xml_handler`
  - `xml-handler-bindings = configuration,dialplan,directory,languages`
- [app/switch/resources/scripts/app.lua](app/switch/resources/scripts/app.lua)
  - 路由到 [app/switch/resources/scripts/app/xml_handler/index.lua](app/switch/resources/scripts/app/xml_handler/index.lua)
- [app/switch/resources/scripts/app/xml_handler/index.lua](app/switch/resources/scripts/app/xml_handler/index.lua)
  - 按 `XML_REQUEST[section]` 分发到：
    - `configuration/*`
    - `directory/*`
    - `dialplan/*`
    - `languages/*`

说明：这是 **FreeSWITCH 向 FusionPBX “拉取”配置** 的机制。

### 3.2 CDR 上报

FreeSWITCH `mod_xml_cdr` 支持两种方式进入 FusionPBX：

1. **HTTP POST** 到 [app/xml_cdr/xml_cdr_import.php](app/xml_cdr/xml_cdr_import.php)
2. **落盘** 到日志目录后由 `xml_cdr_service` 监控导入

相关文件：

- [app/switch/resources/conf/autoload_configs/xml_cdr.conf.xml](app/switch/resources/conf/autoload_configs/xml_cdr.conf.xml)
- [app/xml_cdr/xml_cdr_import.php](app/xml_cdr/xml_cdr_import.php)
- [app/xml_cdr/resources/classes/xml_cdr_service.php](app/xml_cdr/resources/classes/xml_cdr_service.php)

---

## 4) 一句话总结

- **控制面（下行）**：FusionPBX 通过 `event_socket::api/command/bgapi` 调 FreeSWITCH。
- **实时事件（上行）**：FreeSWITCH 事件经 ESL 被 FusionPBX 服务订阅，再通过 WebSocket 推到前端。
- **配置/话单（上行）**：配置通过 Lua XML Handler 动态拉取，CDR 通过 `mod_xml_cdr` HTTP/文件方式导入。

---

## 5) 端到端时序（文字版）

### 场景 A：前端触发“控制动作”（例如会议静音/踢人）

1. 浏览器通过 WebSocket 向服务发送 `topic=action`（服务名如 `active.conferences`）。
2. WebSocket 中心服务（[core/websockets/resources/classes/websocket_service.php](core/websockets/resources/classes/websocket_service.php)）鉴权后，把消息路由给 `active_conferences_service`。
3. `active_conferences_service::handle_action()` 校验权限与参数。
4. 服务调用 `event_socket::api("conference ...")` 或 `event_socket::api("uuid_kill ...")`。
5. FreeSWITCH 执行命令并返回结果。
6. 服务组装 `action_response`，经 WebSocket 返回浏览器。

关键文件：

- [core/websockets/resources/classes/websocket_service.php](core/websockets/resources/classes/websocket_service.php)
- [app/active_conferences/resources/classes/active_conferences_service.php](app/active_conferences/resources/classes/active_conferences_service.php)
- [resources/classes/event_socket.php](resources/classes/event_socket.php)

### 场景 B：FreeSWITCH 实时事件回推到页面（例如来电状态变更）

1. `active_calls_service` 启动并连接 ESL。
2. 发送 `event plain all` + 多条 `filter`（仅保留关心事件）。
3. FreeSWITCH 产生事件（如 `CHANNEL_CREATE/CHANNEL_CALLSTATE/CHANNEL_DESTROY`）。
4. 服务侧 `read_event()` 收到原始事件并转换为 `event_message`。
5. 服务把消息发送给 WebSocket 中心服务（服务名 `active.calls`）。
6. WebSocket 中心按订阅关系和权限过滤后广播给浏览器。
7. 前端刷新坐席/通话状态。

关键文件：

- [app/active_calls/resources/classes/active_calls_service.php](app/active_calls/resources/classes/active_calls_service.php)
- [core/websockets/resources/classes/websocket_service.php](core/websockets/resources/classes/websocket_service.php)
- [resources/classes/event_socket.php](resources/classes/event_socket.php)

### 场景 C：通话结束后 CDR 入库

1. FreeSWITCH `mod_xml_cdr` 生成 CDR。
2. 路径 1：HTTP POST 到 [app/xml_cdr/xml_cdr_import.php](app/xml_cdr/xml_cdr_import.php)。
3. 路径 2：落盘到 `xml_cdr` 目录，由 `xml_cdr_service` 监控并导入。
4. `xml_cdr` 类处理并写入数据库，供报表页面查询。

关键文件：

- [app/switch/resources/conf/autoload_configs/xml_cdr.conf.xml](app/switch/resources/conf/autoload_configs/xml_cdr.conf.xml)
- [app/xml_cdr/xml_cdr_import.php](app/xml_cdr/xml_cdr_import.php)
- [app/xml_cdr/resources/classes/xml_cdr_service.php](app/xml_cdr/resources/classes/xml_cdr_service.php)

---

## 6) 常见故障排查点（按链路）

### ESL 调用失败（FusionPBX -> FreeSWITCH）

- 检查 `config.conf` 中 `switch.event_socket.host/port/password`。
- 检查 FreeSWITCH `event_socket` 是否监听、ACL 是否允许 FusionPBX 主机连接。
- 观察 `event_socket::connect()` 是否认证成功（`+OK accepted`）。

### 事件不更新（FreeSWITCH -> 页面）

- 确认 `active_calls` / `active_conferences` 服务进程在运行。
- 确认已执行 `event plain all` 且 `filter` 返回 `+OK filter added`。
- 确认 [core/websockets/resources/service/websockets.php](core/websockets/resources/service/websockets.php) 进程正常，且服务端与浏览器鉴权成功。

### 页面有请求但无权限数据

- 检查 WebSocket 订阅 token 是否有效。
- 检查权限映射过滤（如 `PERMISSION_MAP`、`create_filter_chain_for`）。

### CDR 缺失

- 若走 HTTP：检查 [app/xml_cdr/xml_cdr_import.php](app/xml_cdr/xml_cdr_import.php) 可访问、来源 IP 命中 CIDR 白名单。
- 若走落盘：检查 [app/xml_cdr/resources/classes/xml_cdr_service.php](app/xml_cdr/resources/classes/xml_cdr_service.php) 对应服务是否运行、目录权限与 inotify 可用性。

---

## 7) 前端截图模块 -> 文件路径映射

> 说明：本节按截图中的卡片名称，整理其在代码中的 **Dashboard 配置文件**、**Widget 渲染文件**（由 `widget_path` 解析）、以及 **点击跳转入口**（`widget_url`）。

### 7.1 Dashboard 渲染机制（路径如何对接）

[core/dashboard/index.php](core/dashboard/index.php) 会读取每个卡片的 `widget_path`，再按下面规则定位渲染文件：

1. `widget_path` 形如 `app_name/widget_name`
2. 解析后拼接：`*/{app_name}/resources/dashboard/{widget_name}.php`
3. `include` 该文件完成卡片内容渲染

关键代码：

- [core/dashboard/index.php](core/dashboard/index.php)（`widget_path` 解析与 include）

---

### 7.2 截图中各功能模块映射表

| 前端模块 | Dashboard 配置文件 | Widget 渲染文件（widget_path） | 点击跳转入口（widget_url） | 主要功能入口文件 |
|---|---|---|---|---|
| Call Detail Records | [app/xml_cdr/resources/dashboard/config.php](app/xml_cdr/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php)（`dashboard/icon`） | `/app/xml_cdr/xml_cdr.php` | [app/xml_cdr/xml_cdr.php](app/xml_cdr/xml_cdr.php) |
| Conference Centers | [app/conference_centers/resources/dashboard/config.php](app/conference_centers/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/conference_centers/conference_rooms.php` | [app/conference_centers/conference_rooms.php](app/conference_centers/conference_rooms.php) |
| Conferences | [app/conferences/resources/dashboard/config.php](app/conferences/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/conferences/conferences.php` | [app/conferences/conferences.php](app/conferences/conferences.php) |
| Contacts | [core/contacts/resources/dashboard/config.php](core/contacts/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/core/contacts/contacts.php` | [core/contacts/contacts.php](core/contacts/contacts.php) |
| Destinations | [app/destinations/resources/dashboard/config.php](app/destinations/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/destinations/destinations.php` | [app/destinations/destinations.php](app/destinations/destinations.php) |
| Devices | [app/devices/resources/dashboard/config.php](app/devices/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/devices/devices.php` | [app/devices/devices.php](app/devices/devices.php) |
| Event Guard | [app/event_guard/resources/dashboard/config.php](app/event_guard/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/event_guard/event_guard_logs.php` | [app/event_guard/event_guard_logs.php](app/event_guard/event_guard_logs.php) |
| Extensions | [app/extensions/resources/dashboard/config.php](app/extensions/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/extensions/extensions.php` | [app/extensions/extensions.php](app/extensions/extensions.php) |
| IVR Menus | [app/ivr_menus/resources/dashboard/config.php](app/ivr_menus/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/ivr_menus/ivr_menus.php` | [app/ivr_menus/ivr_menus.php](app/ivr_menus/ivr_menus.php) |
| Recordings | [app/recordings/resources/dashboard/config.php](app/recordings/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/recordings/recordings.php` | [app/recordings/recordings.php](app/recordings/recordings.php) |
| Ring Groups | [app/ring_groups/resources/dashboard/config.php](app/ring_groups/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php)（配置中存在历史写法） | `/app/ring_groups/ring_groups.php` | [app/ring_groups/ring_groups.php](app/ring_groups/ring_groups.php) |
| Time Conditions | [app/time_conditions/resources/dashboard/config.php](app/time_conditions/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/time_conditions/time_conditions.php` | [app/time_conditions/time_conditions.php](app/time_conditions/time_conditions.php) |
| User Profile | [core/users/resources/dashboard/config.php](core/users/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/core/users/user_profile.php` | [core/users/user_profile.php](core/users/user_profile.php) |
| Users | [core/users/resources/dashboard/config.php](core/users/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/core/users/users.php` | [core/users/users.php](core/users/users.php) |
| Voicemails | [app/voicemails/resources/dashboard/config.php](app/voicemails/resources/dashboard/config.php) | [core/dashboard/resources/dashboard/icon.php](core/dashboard/resources/dashboard/icon.php) | `/app/voicemails/voicemails.php` | [app/voicemails/voicemails.php](app/voicemails/voicemails.php) |
| Domains | [core/domains/resources/dashboard/config.php](core/domains/resources/dashboard/config.php) | [core/domains/resources/dashboard/domains.php](core/domains/resources/dashboard/domains.php)（`domains/domains`） | `/core/domains/domains.php` | [core/domains/domains.php](core/domains/domains.php) |
| Missed Calls | [app/xml_cdr/resources/dashboard/config.php](app/xml_cdr/resources/dashboard/config.php) | [app/xml_cdr/resources/dashboard/missed_calls.php](app/xml_cdr/resources/dashboard/missed_calls.php)（`xml_cdr/missed_calls`） | （空，标题点击跳 `xml_cdr` 过滤页） | [app/xml_cdr/xml_cdr.php](app/xml_cdr/xml_cdr.php)（`call_result=missed`） |
| New Messages | [app/voicemails/resources/dashboard/config.php](app/voicemails/resources/dashboard/config.php) | [app/voicemails/resources/dashboard/voicemails.php](app/voicemails/resources/dashboard/voicemails.php)（`voicemails/voicemails`） | （空，标题点击跳消息页） | [app/voicemails/voicemail_messages.php](app/voicemails/voicemail_messages.php) |
| Recent Calls | [app/xml_cdr/resources/dashboard/config.php](app/xml_cdr/resources/dashboard/config.php) | [app/xml_cdr/resources/dashboard/recent_calls.php](app/xml_cdr/resources/dashboard/recent_calls.php)（`xml_cdr/recent_calls`） | （空，标题点击跳 CDR 页） | [app/xml_cdr/xml_cdr.php](app/xml_cdr/xml_cdr.php) |
| Registrations | [app/registrations/resources/dashboard/config.php](app/registrations/resources/dashboard/config.php) | [app/registrations/resources/dashboard/registrations.php](app/registrations/resources/dashboard/registrations.php)（`registrations/registrations`） | `/app/registrations/registrations.php` | [app/registrations/registrations.php](app/registrations/registrations.php) |

---

### 7.3 补充说明（截图中“计数型卡片”）

- `Missed Calls / Recent Calls / New Messages` 这三个卡片在配置里 `widget_url` 为空，主要通过对应的 widget 渲染文件输出数据和标题链接。
- `Domains / Registrations` 属于“图标+数字徽标”卡片，数字由各自 widget 文件实时查询生成。
- 如果你要新增同类卡片，通常只需：
  1) 在对应应用新增 `resources/dashboard/config.php` 配置项；
  2) 提供 `resources/dashboard/{widget_name}.php` 渲染文件；
  3) 设置 `widget_url`（若需要直接跳转）。
