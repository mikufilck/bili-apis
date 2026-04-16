# 直播间信息流

## 获取信息流认证秘钥

> https://api.live.bilibili.com/xlive/web-room/v1/index/getDanmuInfo

*请求方式：GET*

**url参数：**

| 参数名 | 类型 | 内容         | 必要性 | 备注 |
| ------ | ---- | ------------ | ------ | ---- |
| id     | num  | 直播间真实id | 必要   |      |

**json回复：**

根对象：

| 字段    | 类型 | 内容     | 备注                                                         |
| ------- | ---- | -------- | ------------------------------------------------------------ |
| code    | num  | 返回值   | 0：成功<br />65530：token错误（登录错误）<br />1：错误<br />60009：分区不存在<br />**（其他错误码有待补充）** |
| message | str  | 错误信息 | 默认为空                                                     |
| ttl     | num  | 1        |                                                              |
| data    | obj  | 信息本体 |                                                              |

`data`对象：

| 字段               | 类型  | 内容                 | 备注 |
| ------------------ | ----- | ------------------- | ---- |
| group              | str   | live                |      |
| business_id        | num   | 0                   |      |
| refresh_row_factor | num   | 0.125               |      |
| refresh_rate       | num   | 100                 |      |
| max_delay          | num   | 5000                |      |
| token              | str   | 认证秘钥            |      |
| host_list          | array | 信息流服务器节点列表 |      |

`host_list`数组中的对象：

| 字段     | 类型 | 内容       | 备注 |
| -------- | ---- | ---------- | ---- |
| host     | str  | 服务器域名 |      |
| port     | num  | tcp端口    |      |
| wss_port | num  | wss端口    |      |
| ws_port  | num  | ws端口     |      |


**示例：**

获得直播间`22824550`的信息流认证秘钥

```shell
curl -G 'https://api.live.bilibili.com/xlive/web-room/v1/index/getDanmuInfo' \
--data-urlencode 'id=22824550'
```

<details>
<summary>查看响应示例：</summary>

```json
{
  "code": 0,
  "message": "0",
  "ttl": 1,
  "data": {
    "group": "live",
    "business_id": 0,
    "refresh_row_factor": 0.125,
    "refresh_rate": 100,
    "max_delay": 5000,
    "token": "Eac3Lm1JADzny-YnB5MW0MQcd23rw_mgMFZAnu40I-J2ecP2Qj6CH-UqjdfvwiqVEZcEksG1ONSOi1dGzm0wM4FxqA-ZYXtcQyHXPXqxmrx3AmDx8Z5-d4TuKQkaU0zxevH1B-gnu7g8TDtIE4lns4BYlw==",
    "host_list": [
      {
        "host": "tx-sh-live-comet-02.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      },
      {
        "host": "tx-bj-live-comet-02.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      },
      {
        "host": "broadcastlv.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      }
    ]
  }
}
```

</details>

**注:最终URI格式为： host+对应port+"/sub"**，例如以上示例中一个可行的ws连接URI应当为`tx-sh-live-comet-02.chat.bilibili.com:2244/sub`


## 数据包格式

数据包为MQ（Message Queue，消息队列）使用Websocket或TCP连接作为通道，具体格式为头部数据+正文数据

操作流程：

发送认证包->接收认证包回应->接收普通包&（每30秒发送心跳包->接收心跳回应）

头部格式：

| 偏移量 | 长度 | 类型   | 含义                                                         |
| ------ | ---- | ------ | ------------------------------------------------------------ |
| 0      | 4    | uint32 | 封包总大小（头部大小+正文大小）                              |
| 4      | 2    | uint16 | 头部大小（一般为0x0010，16字节）                             |
| 6      | 2    | uint16 | 协议版本:<br />0普通包正文不使用压缩 <br />1心跳及认证包正文不使用压缩<br />2普通包正文使用zlib压缩<br/>3普通包正文使用brotli压缩,解压为一个带头部的协议0普通包 |
| 8      | 4    | uint32 | 操作码（封包类型）                                           |
| 12     | 4    | uint32 | sequence，每次发包时向上递增                                 |

操作码：

| 代码 | 含义                 |
| ---- | -------------------- |
| 2    | 心跳包               |
| 3    | 心跳包回复（人气值） |
| 5    | 普通包（命令）       |
| 7    | 认证包               |
| 8    | 认证包回复           |

*普通包可能包含多条命令，每个命令有一个头部，指示该条命令的长度等信息*

## 数据包

### 认证包

方式：（上行）

连接成功后5秒内发送，否则强制断开连接

正文：

json格式

| 字段     | 类型 | 内容         | 必要性 | 备注               |
| -------- | ---- | ------------ | ------ | ------------------ |
| uid      | num  | 用户mid      | 非必要 | uid为0即为游客登录 |
| roomid   | num  | 加入房间的id | 必要   | 直播间真实id       |
| protover | num  | 协议版本     | 非必要 | 3                  |
| platform | str  | 平台标识     | 非必要 | "web"              |
| type     | num  | 2            | 非必要 |                    |
| key      | str  | 认证秘钥     | 非必要 |                    |

示例：

```
00000000: 0000 00ff 0010 0001 0000 0007 0000 0001  ................
00000001: 7b22 7569 6422 3a31 3630 3134 3836 3234  {"uid":160148624
00000002: 2c22 726f 6f6d 6964 223a 3232 3630 3831  ,"roomid":226081
00000003: 3132 2c22 7072 6f74 6f76 6572 223a 332c  12,"protover":3,
00000004: 2270 6c61 7466 6f72 6d22 3a22 7765 6222  "platform":"web"
00000005: 2c22 7479 7065 223a 322c 226b 6579 223a  ,"type":2,"key":
00000006: 2230 7670 5448 5737 7757 556e 6c6f 5270  "0vpTHW7wWUnloRp
00000007: 5251 6b47 764e 626e 7776 7364 6d2d 7159  RQkGvNbnwvsdm-qY
00000008: 4777 4243 5875 2d59 5164 6e57 7653 5547  GwBCXu-YQdnWvSUG
00000009: 7373 4139 7962 4b68 7932 6a78 3952 6f63  ssA9ybKhy2jx9Roc
0000000a: 4150 4651 6d54 4f6b 5277 6b4b 687a 4479  APFQmTOkRwkKhzDy
0000000b: 4839 5054 756f 5468 6834 4630 7562 584c  H9PTuoThh4F0ubXL
0000000c: 4964 6e69 3734 5539 304b 4242 6972 3248  Idni74U90KBBir2H
0000000d: 7451 3941 3777 674b 3438 4b7a 495f 5a5a  tQ9A7wgK48KzI_ZZ
0000000e: 3838 7557 4e59 6652 4f48 6964 4e6a 3732  88uWNYfROHidNj72
0000000f: 7061 796e 3479 3071 4268 513d 3d22 7d    payn4y0qBhQ=="}
```



### 认证包回复

方式：（下行）

在认证包发送成功后就会收到

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| code | num  | 返回值 | 0认证成功 |

示例：

```
00000000: 0000 001a 0010 0001 0000 0008 0000 0001  ................
00000001: 7b22 636f 6465 223a 307d                 {"code":0}
```



### 心跳包

方式：（上行）

30秒左右发送一次，否则60秒后会被强制断开连接

正文：

可以为空或任意字符

示例：

```
00000000: 0000 001f 0010 0001 0000 0002 0000 0001  ................
00000001: 5b6f 626a 6563 7420 4f62 6a65 6374 5d    [object Object]
```

### 心跳包回复（人气值）

方式：（下行）

在心跳包发送成功后就会收到

正文：

正文分为两个部分，第一部分是人气值 [uint32整数，代表房间当前的人气值]

第二部分是对于心跳包内容的复制，心跳包正文是什么这里就会回应什么。

示例：

```
00000000: 0000 0014 0010 0001 0000 0003 0000 0000  ................
00000001: 0000 09a2 5b6f 626a 6563 7420 4f62 6a65  ....[object Obje
00000002: 6374 5d                                  ct]
```

可见房间内人气值为2466（0x000009a2）

### 普通包

方式：（下行）

正文：

正文一般为普通JSON数据。

大多数普通包都经过zlib压缩或brotli压缩。

示例：

```
00000000: 0000 0086 0010 0003 0000 0005 0000 0000  ................
00000001: 8b38 8000 0000 7200 1000 0000 0000 0500  .8....r.........
00000002: 0000 007b 2263 6d64 223a 2257 4154 4348  ...{"cmd":"WATCH
00000003: 4544 5f43 4841 4e47 4522 2c22 6461 7461  ED_CHANGE","data
00000004: 223a 7b22 6e75 6d22 3a32 3230 3937 2c22  ":{"num":22097,"
00000005: 7465 7874 5f73 6d61 6c6c 223a 2232 2e32  text_small":"2.2
00000006: e4b8 8722 2c22 7465 7874 5f6c 6172 6765  ...","text_large
00000007: 223a 2232 2e32 e4b8 87e4 baba e79c 8be8  ":"2.2..........
00000008: bf87 227d 7d03                           .."}}.
```

---

- [弹幕](#弹幕)
- [进场关注或分享消息](#进场关注或分享消息)
- [用户庆祝消息](#用户庆祝消息)
- [醒目留言](#醒目留言)
- [醒目留言日文翻译](#醒目留言日文翻译)
- [送礼](#送礼)
- [礼物星球点亮](#礼物星球点亮)
- [通用系统广播弹幕](#通用系统广播弹幕)
- [礼物连击](#礼物连击)
- [通知消息](#通知消息)
- [主播准备中](#主播准备中)
- [直播开始](#直播开始)
- [主播信息更新](#主播信息更新)
- [直播间高能榜](#直播间高能榜)
- [直播间高能用户数量](#直播间高能用户数量)
- [用户到达直播间高能榜前三名的消息](#用户到达直播间高能榜前三名的消息)
- [PK 礼物积分同步](#PK礼物积分同步)
- [PK 礼物积分同步V2](#PK礼物积分同步V2)
- [PK 战况与 MVP 实时更新](#PK战况与MVP实时更新)
- [PK 全局状态](#PK全局状态)
- [PK 惩罚战斗结束](#PK惩罚战斗结束)
- [PK 战斗结算与惩罚](#PK战斗结算与惩罚)
- [PK 任务进度挂件](#PK任务进度挂件)
- [直播间系统吐司提示](#直播间系统吐司提示)
- [直播间用户点赞](#直播间用户点赞)
- [直播间点赞数](#直播间点赞数)
- [直播间发红包弹幕](#直播间发红包弹幕)
- [直播间红包](#直播间红包)
- [直播间抢到红包的用户](#直播间抢到红包的用户)
- [天选时刻开始](#天选时刻开始)
- [天选时刻弹幕聚合](#天选时刻弹幕聚合)
- [天选时刻开奖](#天选时刻开奖)
- [直播间看过人数](#直播间看过人数)
- [用户进场特效](#用户进场特效)
- [主播排行榜变动](#主播排行榜变动)
- [榜单排名刷新](#榜单排名刷新)
- [人气排行标签页变更](#人气排行标签页变更)
- [直播间在所属分区的排名改变](#直播间在所属分区的排名改变)
- [直播间在所属分区排名提升的祝福](#直播间在所属分区排名提升的祝福)
- [直播间信息更改](#直播间信息更改)
- [醒目留言按钮状态](#醒目留言按钮状态)
- [互动游戏状态变更](#互动游戏状态变更)
- [开放平台互动游戏配置大包](#开放平台互动游戏配置大包)
- [互动游戏按钮状态变更](#互动游戏按钮状态变更)
- [直播间控制面板动态配置](#直播间控制面板动态配置)
- [礼物面板动态配置](#礼物面板动态配置)
- [顶部横幅](#顶部横幅)
- [下播的直播间](#下播的直播间)
- [未知消息](#未知消息)

---


#### 弹幕

当收到弹幕时接收到此条消息

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str  | "DANMU_MSG" | 如果是弹幕消息，内容则是"DANMU_MSG" |
| dm_v2 | str  | 待调查 | 目前观察到为空字符串 |
| info | array | 这条弹幕的用户、内容与粉丝勋章等各种信息 | 待调查其中每个数据的含义 |

info字段

| 索引 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| 0 | array | 弹幕信息 | |
| 1 | str | 弹幕文本 | |
| 2 | array | 发送者信息 | |
| 3 | array | 发送者粉丝勋章信息 | 若没有粉丝勋章则为空数组 |
| 4 | array | 发送者UL等级信息 | |
| 5 | array | 待调查 | |
| 6 | num | 待调查 | |
| 7 | num | 待调查 | |
| 8 | obj | 待调查 | 有时为null |
| 9 | obj | 弹幕发送的Unix时间戳 | 包含ts和ct字段 |
| 10 | num | 待调查 | |
| 11 | num | 待调查 | |
| 12 | obj | 待调查 | 有时为null |
| 13 | obj | 待调查 | 有时为null |
| 14 | num | 待调查 | |
| 15 | num | 待调查 | 旧版常见值为21，新版观察到有208等 |
| 16 | array | 待调查 | |
| 17 | obj | 待调查 | 有时为null |

9索引

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| ts | num | 弹幕发送的时间戳 | 精确到秒 |
| ct | str | 校验码或特征码 | 16进制字符串 |

0索引

| 索引 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| 0 | num | 待调查 | |
| 1 | num | 弹幕模式 | 弹幕的mode字段 |
| 2 | num | 弹幕字体大小 | 弹幕的fontsize字段 |
| 3 | num | 弹幕颜色 | 弹幕的color字段<br />十六进制颜色值的十进制数字 |
| 4 | num | 待调查 | 弹幕发送时的Unix时间戳，精确到毫秒 |
| 5 | num | 弹幕发送时的Unix时间戳 | 弹幕的rnd字段，精确到秒 |
| 6 | num | 待调查 | |
| 7 | str | 用户Hash | 用户的特征码/哈希码 |
| 8 | num | 待调查 | |
| 9 | num | 待调查 | |
| 10 | num | 待调查 | |
| 11 | str | 待调查 | |
| 12 | num | 待调查 | |
| 13 | str | 待调查 | |
| 14 | str | 待调查 | |
| 15 | obj | 弹幕扩展与用户详情 | |
| 16 | obj | 活动信息 | |
| 17 | num | 待调查 | |

2索引

| 索引 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| 0 | num | 发送者UID | |
| 1 | str | 发送者昵称 | |
| 2 | num | 待调查 | |
| 3 | num | 待调查 | |
| 4 | num | 待调查 | |
| 5 | num | 待调查 | 通常为10000 |
| 6 | num | 待调查 | 通常为1 |
| 7 | str | 待调查 | |

0索引的15索引

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| extra | str | 弹幕额外信息 | 包含大量弹幕UI展示参数的JSON字符串（需反序列化解析，结构见下方） |
| mode | num | 待调查 | |
| show_player_type | num | 待调查 | |
| user | obj | 发送者详细信息 | |

0索引的15索引的extra字段 (反序列化后的内部JSON对象)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| send_from_me | bool | 是否由自己发送 | |
| master_player_hidden | bool | 是否隐藏房主标志 | |
| mode | num | 弹幕模式 | |
| color | num | 弹幕颜色十进制值 | |
| dm_type | num | 弹幕类型 | |
| font_size | num | 字体大小 | |
| player_mode | num | 播放器模式 | |
| show_player_type | num | 显示播放器类型 | |
| content | str | 弹幕文本内容 | 与1索引内容一致 |
| user_hash | str | 用户Hash | 与0索引的7索引一致 |
| emoticon_unique | str | 表情包唯一标识 | |
| bulge_display | num | 突出显示标志 | |
| recommend_score | num | 推荐分数 | |
| dm_score | num | 弹幕分数 | |
| chronos_force_display | num | 待调查 | |
| main_state_dm_color | str | 主状态弹幕颜色 | |
| objective_state_dm_color| str | 客观状态弹幕颜色| |
| direction | num | 方向 | |
| pk_direction | num | PK方向 | |
| quartet_direction | num | 待调查 | |
| anniversary_crowd | num | 周年人群标志 | |
| yeah_space_type | str | 待调查 | |
| yeah_space_url | str | 待调查 | |
| jump_to_url | str | 跳转链接 | |
| space_type | str | 空间类型 | |
| space_url | str | 空间链接 | |
| animation | obj | 动画参数 | |
| emots | obj | 表情数据 | 若无则为null |
| is_audited | bool | 是否经过审核 | |
| id_str | str | 弹幕唯一字符串ID | 常用于精准撤回弹幕 |
| icon | obj | 弹幕图标 | 若无则为null |
| show_reply | bool | 是否允许回复 | |
| reply_mid | num | 回复目标UID | 0表示未回复任何人 |
| reply_uname | str | 回复目标昵称 | |
| reply_uname_color | str | 回复目标昵称颜色 | |
| reply_is_mystery | bool | 回复目标是否神秘人| |
| reply_type_enum | num | 回复类型枚举 | |
| hit_combo | num | 连击数 | |
| esports_jump_url | str | 电竞跳转链接 | |
| is_mirror | bool | 是否镜像 | |
| is_collaboration_member | bool | 是否为合作成员 | |
| card | obj | 卡片信息 | 见下方展开 |
| voice | obj | 语音信息 | 若无则为null |
| background_type | num | 背景类型 | |

0索引的15索引的extra的card字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| card_type | num | 卡片类型 | |
| oid_str | str | 待调查 | |
| oid_str_1 | str | 待调查 | |
| origin_oid_str | str | 待调查 | |
| share_id | str | 分享ID | |
| share_origin | str | 分享来源 | |
| from | str | 来源 | |
| card_content | obj | 卡片内容 | 若无则为null |

0索引的15索引的user字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| base | obj | 基础资料 | 见下方展开 |
| guard | obj | 舰队信息 | 若无则为null |
| guard_leader | obj | 舰队队长状态 | 见下方展开 |
| medal | obj | 粉丝勋章详情 | 若无则为null |
| title | obj | 称号信息 | 见下方展开 |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| uid | num | 发送者UID | |
| wealth | obj | 财富信息 | 若无则为null |

0索引的15索引的user的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 头像链接 | |
| is_mystery | bool | 是否神秘人 | |
| name | str | 用户昵称 | |
| name_color | num | 昵称颜色 | |
| name_color_str | str | 昵称颜色字符串 | |
| official_info | obj | 官方认证信息 | 见下方展开 |
| origin_info | obj | 原始信息 | 见下方展开 |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |

0索引的15索引的user的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| desc | str | 认证描述 | |
| role | num | 认证角色 | |
| title | str | 认证头衔 | |
| type | num | 认证类型 | |

0索引的15索引的user的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 原始头像链接 | |
| name | str | 原始昵称 | |

0索引的15索引的user的guard_leader字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| is_guard_leader | bool | 是否舰队队长 | |

0索引的15索引的user的title字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| old_title_css_id | str | 旧版称号CSS的ID | |
| title_css_id | str | 称号CSS的ID | |

0索引的16索引

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| activity_identity | str | 待调查 | |
| activity_source | num | 待调查 | |
| not_show | num | 待调查 | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "DANMU_MSG",
  "dm_v2": "",
  "info": [
    [
      0,
      1,
      25,
      16777215,
      1775751629449,
      1775751630,
      0,
      "70d2585a",
      0,
      0,
      0,
      "",
      0,
      "{}",
      "{}",
      {
        "extra": "{\"send_from_me\":false,\"master_player_hidden\":false,\"mode\":0,\"color\":16777215,\"dm_type\":0,\"font_size\":25,\"player_mode\":1,\"show_player_type\":0,\"content\":\"欢迎 mikufilck 来到直播间！\",\"user_hash\":\"1892833370\",\"emoticon_unique\":\"\",\"bulge_display\":0,\"recommend_score\":0,\"dm_score\":0,\"chronos_force_display\":0,\"main_state_dm_color\":\"\",\"objective_state_dm_color\":\"\",\"direction\":0,\"pk_direction\":0,\"quartet_direction\":0,\"anniversary_crowd\":0,\"yeah_space_type\":\"\",\"yeah_space_url\":\"\",\"jump_to_url\":\"\",\"space_type\":\"\",\"space_url\":\"\",\"animation\":{},\"emots\":null,\"is_audited\":false,\"id_str\":\"51f8de4ff040fc4f5d9bc5d30069d7d16508\",\"icon\":null,\"show_reply\":true,\"reply_mid\":0,\"reply_uname\":\"\",\"reply_uname_color\":\"\",\"reply_is_mystery\":false,\"reply_type_enum\":0,\"hit_combo\":0,\"esports_jump_url\":\"\",\"is_mirror\":false,\"is_collaboration_member\":false,\"card\":{\"card_type\":0,\"oid_str\":\"\",\"oid_str_1\":\"\",\"origin_oid_str\":\"\",\"share_id\":\"\",\"share_origin\":\"\",\"from\":\"\",\"card_content\":null},\"voice\":null,\"background_type\":0}",
        "mode": 0,
        "show_player_type": 0,
        "user": {
          "base": {
            "face": "[https://i0.hdslb.com/bfs/face/0348ddae8ea088ab5f38aaeb4842fb57968b86de.jpg](https://i0.hdslb.com/bfs/face/0348ddae8ea088ab5f38aaeb4842fb57968b86de.jpg)",
            "is_mystery": false,
            "name": "ACGN_mikufilck",
            "name_color": 0,
            "name_color_str": "",
            "official_info": {
              "desc": "",
              "role": 0,
              "title": "",
              "type": -1
            },
            "origin_info": {
              "face": "[https://i0.hdslb.com/bfs/face/0348ddae8ea088ab5f38aaeb4842fb57968b86de.jpg](https://i0.hdslb.com/bfs/face/0348ddae8ea088ab5f38aaeb4842fb57968b86de.jpg)",
              "name": "ACGN_mikufilck"
            },
            "risk_ctrl_info": null
          },
          "guard": null,
          "guard_leader": {
            "is_guard_leader": false
          },
          "medal": null,
          "title": {
            "old_title_css_id": "",
            "title_css_id": ""
          },
          "uhead_frame": null,
          "uid": 3493286400494454,
          "wealth": null
        }
      },
      {
        "activity_identity": "",
        "activity_source": 0,
        "not_show": 0
      },
      0
    ],
    "欢迎 mikufilck 来到直播间！",
    [
      3493286400494454,
      "ACGN_mikufilck",
      0,
      0,
      0,
      10000,
      1,
      ""
    ],
    [],
    [
      0,
      0,
      9868950,
      ">50000",
      2
    ],
    [
      "",
      ""
    ],
    0,
    0,
    null,
    {
      "ct": "5814F6B",
      "ts": 1775751629
    },
    0,
    0,
    null,
    null,
    0,
    208,
    [
      7
    ],
    null
  ]
}
```

</details>


#### 连续弹幕消息

连续多条相同弹幕，或者分享时触发

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str  | "DM_INTERACTION" | 如果是连续弹幕或分享消息，内容是"DM_INTERACTION" |
| data | obj  | 互动状态详情 |  |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| id | num | 事件唯一ID | 前端根据此ID合并或更新同一个 UI 框的状态。 |
| type | num | 互动类型 | `105` 为分享直播间。其他可能为高频弹幕造梗连击等。 |
| status | num | 状态码 | 可能是 UI 框的生命周期（如出现、维持、消散）。 |
| dmscore | num | 互动积分 | |
| data | str | 序列化的 JSON 字符串 | **注意这是一个字符串格式的 JSON，包含具体的 UI 渲染控制参数。** |

data.data字段 (将字符串反序列化后的内部对象)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cnt | num | 累计数量 | 例如：当前累计有 `1` 人触发。 |
| suffix_text | str | 后缀文本 | 例如："人分享了直播间" 或 "x连击" |
| fade_duration | num | UI消散时间 | 单位为毫秒，例如 `10000` 代表停留 10 秒。 |
| display_flag | num | 是否展示标志 | 1展示，0隐藏 |
| card_appear_interval | num | 出现间隔时间 | 待调查 |
| reset_cnt | num | 重置计数 | 待调查 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "DM_INTERACTION",
  "data": {
    "data": "{\"fade_duration\":10000,\"cnt\":1,\"card_appear_interval\":0,\"suffix_text\":\"人分享了直播间\",\"reset_cnt\":0,\"display_flag\":1}",
    "dmscore": 36,
    "id": 158234096820736,
    "status": 4,
    "type": 105
  }
}
```

</details>

#### 进场关注或分享消息

B站目前已将普通用户的互动消息（进场、关注、分享等）全面迁移至 `INTERACT_WORD_V2`。
为了在直播间高并发时极大地压缩数据体积，核心的用户身份信息和互动动作被序列化成了 **Protobuf 二进制流**，并经过 Base64 编码后统一打包在 `pb` 字段中。外层仅保留用于渲染权重的积分字段。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "INTERACT_WORD_V2" | 用户进入、关注、分享直播间时统一触发此指令 |
| data | obj  | 包含展示权重和加密的 pb 数据流 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| dmscore | num | 弹幕积分/展示权重 | 系统根据用户等级和行为价值动态打分。分数越高代表用户越尊贵或行为越重要（如分享/关注的分数通常远高于普通进场）。常用于客户端高并发防刷屏时的丢弃策略与优先渲染。 |
| pb | str | Base64编码的Protobuf数据 | **【核心字段】包含真正的交互详情**。开发者需要先进行 Base64 解码，然后按 Protobuf 结构反序列化，才能拿到用户名、动作类型等真实数据。 |

---

#### `pb` 字段解码后的内部结构 (Protobuf)

开发者解码 `pb` 字段后，会得到以下结构的对象。依靠内部的 `msg_type` 字段，即可判断该用户究竟执行了什么动作。

**核心互动类型 (`msg_type`) 枚举：**
* **`1`** : **进入直播间 (Enter)**。如果数据包含 `relation_tail` (第23号字段)，则代表这是系统下发的特殊提示（如：“近期关注了你”、“经常看直播但未关注”）。
* **`2`** : **关注直播间 (Follow)**。通常伴随庞大的 `uinfo` 数据（包含粉丝牌、舰队等所有详细配置）。
* **`3`** : **分享直播间 (Share)**。用户将直播间分享到其他平台时触发。

反序列化后的内部关键字段对照表：

| 字段编号 (Tag) | 字段名 | 数据类型 | 备注 |
| :--- | :--- | :--- | :--- |
| 1 | `uid` | uint64 | 用户的 UID |
| 2 | `uname` | string | 用户的昵称 |
| 3 | `uname_color` | string | 昵称颜色的十六进制文本 |
| 4 | `identities` | array (uint64) | 用户身份组标识 |
| **5** | **`msg_type`** | **uint64** | **互动类型（1:进场, 2:关注, 3:分享）** |
| 6 | `roomid` | uint64 | 当前直播间的房间 ID |
| 7 | `timestamp` | uint64 | 事件发生的时间戳 (秒级) |
| 8 | `score` | uint64 | 互动分数/权重（与外层dmscore相关） |
| 9 | `fans_medal` | object | 粉丝勋章详情 |
| 12 | `contribution` | object | 贡献度信息 |
| 19 | `contribution_v2` | object | 新版贡献度信息 |
| 20 | `group_medal` | object | 粉丝团勋章简要信息 |
| 21 | `is_mystery` | bool | 是否为神秘人 |
| **22** | **`uinfo`** | **object** | **极其详细的用户资料（包含头像、舰队、实名认证、身份有效期等嵌套结构，高等级用户进场或关注时附带）** |
| **23** | **`relation_tail`** | **object** | **主播关系尾缀（通常在 `msg_type=1` 时附加，用于给主播发送交互提示）** |

---

<details>
<summary>展开查看完整的 .proto 定义文件（包含解密的 uinfo 结构）：</summary>

开发者可将以下代码保存为 `INTERACT_WORD_V2.proto`，并使用 `protoc` 工具生成对应语言的解析类库。相比于其他指令的 JSON，这里的 `uinfo` 被序列化成了深层嵌套的结构。

```protobuf
syntax = "proto3";

// 顶层对象
message InteractWordV2 {
    uint64 uid = 1;
    string uname = 2;
    string uname_color = 3;
    repeated uint64 identities = 4;
    uint64 msg_type = 5;       // 核心：1进场，2关注，3分享
    uint64 roomid = 6;
    uint64 timestamp = 7;
    uint64 score = 8;
    FansMedalInfo fans_medal = 9;
    uint64 is_spread = 10;
    string spread_info = 11;
    ContributionInfo contribution = 12;
    string spread_desc = 13;
    uint64 tail_icon = 14;
    uint64 trigger_time = 15;
    uint64 privilege_type = 16;
    uint64 core_user_type = 17;
    string tail_text = 18;
    ContributionInfoV2 contribution_v2 = 19;
    GroupMedalBrief group_medal = 20;
    bool is_mystery = 21;
    UserInfo uinfo = 22;       // 包含头像及深层信息的对象
    UserAnchorRelation relation_tail = 23;
}

// === Uinfo 深度解剖结构 ===
message UserInfo {
    UserBase base = 1;
    UserMedal medal = 2;
    UserWealth wealth = 3;
    string title = 4;          // 字符串形式的简单称号
    UserGuard guard = 5;
    UserHeadFrame uhead_frame = 6;
    UserGuardLeader guard_leader = 7;
}

message UserBase {
    string face = 1;           // 头像 URL
    string name = 2;           // 昵称
    uint32 is_mystery = 3;     // 是否神秘人
    // 内部可能还嵌套有官方认证 official_verify 等字段
}

message UserMedal {
    string name = 1;           // 粉丝牌名称
    uint32 level = 2;          // 粉丝牌等级
    string color_start = 3;    // 渐变色起始
    string color_end = 4;      // 渐变色结束
    string color_border = 5;   // 边框色
    // 高级徽章颜色配置...
}

message UserGuard {
    uint32 level = 1;          // 舰队等级 (1总督, 2提督, 3舰长)
    string expired_str = 2;    // 【关键隐蔽字段】有效期时间戳字符串，如 "2026-05-31 23:59:59"
}

// 某些拥有特殊排行头衔的用户，会在内层挂载相关标识
message UserTitle {
    string old_title = 1;      // 有时用作文本特效，如 "月榜前3用户" 等特殊头衔标识
    string title = 2;
}
// =========================

message ContributionInfo {
    int64 grade = 1;
}

message ContributionInfoV2 {
    uint64 grade = 1;
    string rank_type = 2;
    string text = 3;
}

message FansMedalInfo {
    int64 target_id = 1;
    int64 medal_level = 2;
    string medal_name = 3;
    int64 medal_color = 4;
    int64 medal_color_start = 5;
    int64 medal_color_end = 6;
    int64 medal_color_border = 7;
    int64 is_lighted = 8;
    int64 guard_level = 9;
    string special = 10;
    int64 icon_id = 11;
    int64 anchor_roomid = 12;
    int64 score = 13;
}

message UserAnchorRelation {
    string tail_icon = 1;
    string tail_guide_text = 2;  // 主播提示文本，如"近期关注了你，多多与TA互动吧"
    uint64 tail_type = 3;
}

message GroupMedalBrief {
    uint64 medal_id = 1;
    string name = 2;
    uint64 is_lighted = 3;
}

实际数据示例如下

{
  "cmd": "INTERACT_WORD_V2",
  "data": {
    "dmscore": 176,
    "pb": "CJ3N6wwSCW1pa3VmaWxjayICAwEoAzDwxdkOOLL/484GQKHWmMPXM0oxCOeVgKuwraYGEBUaCeWBmueMq+eahCDLqGkoy6hpMJK7ygI4y6hpQAFg8MXZDmjsE2IAeKbf5aPU38DSGJoBALIBrgIInc3rDBK9AQoJbWlrdWZpbGNrEkpodHRwczovL2kyLmhkc2xiLmNvbS9iZnMvZmFjZS82YTBkZDM2YmE3ZmExOGU4NGM2NTI3Yzg3YmViYTAyYTVkMmRlM2Y5LmpwZzJXCgltaWt1ZmlsY2sSSmh0dHBzOi8vaTIuaGRzbGIuY29tL2Jmcy9mYWNlLzZhMGRkMzZiYTdmYTE4ZTg0YzY1MjdjODdiZWJhMDJhNWQyZGUzZjkuanBnOgsg////////////ARplCgnlgZrnjKvnmoQQFRjLqGkgkrvKAijLqGkwy6hpSAFQ55WAq7CtpgZg7BN6CSMzRkI0RjY5OYIBCSMzRkI0RjY5OYoBCSMzRkI0RjY5OZIBByNGRkZGRkaaAQkjM0ZCNEY2RTYyALoBAA=="
  }
}
```

</details>

#### 上舰通知

json格式

| 字段 | 类型 | 内容               | 备注                               |
| ---- | ---- |------------------|----------------------------------|
| cmd | str | "GUARD_BUY"      | 用户购买舰长 / 提督 / 总督，内容则是"GUARD_BUY" |
| data | obj | 上舰人uid & 昵称、上舰信息 |                                  |

data字段

| 字段 | 类型  | 内容                       | 备注  |
| ---- |-----|--------------------------|-----|
| uid | num | 用户ID                     |     |
| username | str | 用户名称                     |     |
| guard_level | num | 大航海等级  |  1: 总督 2: 提督 3:舰长     |
| num | num | 数量                       |     |
| price | num | 价值（金瓜子）                      |     |
| gift_id | num | 礼物id                     |     |
| gift_name | str | 礼物名称                     |     |
| start_time | num | 开始时间戳                      |     |
| end_time | num | 结束时间戳                      |     |

<details>

<summary>查看消息示例：</summary>

```json
{
  "cmd": "GUARD_BUY",
  "data": {
    "uid": 14225357,
    "username": "妙妙喵喵妙妙喵O_O",
    "guard_level": 3,
    "num": 1,
    "price": 198000,
    "gift_id": 10003,
    "gift_name": "舰长",
    "start_time": 1677069316,
    "end_time": 1677069316
  }
}
```

</details>

#### 用户庆祝消息

json格式

| 字段 | 类型 | 内容               | 备注                               |
| ---- | ---- |------------------|----------------------------------|
| cmd | str | "USER_TOAST_MSG"      | 用户购买舰长 / 提督 / 总督后的庆祝消息，内容包含用户陪伴天数 |
| data | obj | 上舰人uid & 昵称、上舰信息 |                                  |

data字段

| 字段 | 类型  | 内容                       | 备注  |
| ---- |-----|--------------------------|-----|
| anchor_show | bool | 是否显示 |     |
| color | str | 颜色 |     |
| dmscore | num | 待调查  |     |
| effect_id | num | 待调查 |     |
| face_effect_id | num | 待调查  |     |
| gift_id | num | 礼物id |     |
| group_name | str | 待调查  |     |
| group_op_type | num | 待调查   |     |
| group_role_name | str | 待调查  |     |
| guard_level | num | 大航海等级       | 1: 总督 2: 提督 3:舰长 |
| is_group | num | 待调查 |     |
| is_show | num | 待调查 |     |
| num | num | 上舰个数  |     |
| op_type | num | 待调查 |     |
| payflow_id | str | 待调查 |     |
| price | num | 价格 |
| role_name | str | 身份名称 |     |
| room_effect_id | num | 待调查 |     |
| room_group_effect_id | num | 待调查 |     |
| start_time | num | 待调查 |     |
| svga_block | num | 待调查 |     |
| target_guard_count | str | 庆祝消息正文 |     |
| toast_msg | num | 待调查 |     |
| uid | num | 上舰人UID |     |
| unit | str | 购买身份时间单位 |     |
| user_show | bool | 待调查 |     |
| username | str | 上舰人用户名 |     |

<details>

<summary>查看消息示例：</summary>

```json
{
    'anchor_show': True,
    'color': '#00D1F1',
    'dmscore': 90,
    'effect_id': 397,
    'end_time': 1702580687,
    'face_effect_id': 44,
    'gift_id': 10003,
    'group_name': '',
    'group_op_type': 0,
    'group_role_name': '',
    'guard_level': 3,
    'is_group': 0,
    'is_show': 0,
    'num': 1,
    'op_type': 1,
    'payflow_id':'2312150304155852173446521',
    'price': 138000,
    'role_name': '舰长',
    'room_effect_id': 590,
    'room_group_effect_id': 1337,
    'start_time': 1702580687,
    'svga_block': 0,
    'target_guard_count': 146,
    'toast_msg': '<%无光之日%> 在主播Mia米娅-的直播间开通了舰长，今天是TA陪伴主播的第1天',
    'uid': 79667344,
    'unit': '月',
    'user_show': True,
    'username': '无光之日'}
```

</details>

#### 醒目留言

当有人发送醒目留言 (Super Chat) 时接收到此消息

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str  | "SUPER_CHAT_MESSAGE" | 如果是醒目留言，内容则是"SUPER_CHAT_MESSAGE" |
| data | obj | SC详细信息 | 见下方展开 |
| is_report | bool | 待调查 | |
| msg_id | str | 消息唯一ID | 格式通常为时间戳与随机数的组合组合拼接 |
| p_is_ack | bool | 待调查 | |
| p_msg_type | num | 待调查 | |
| send_time | num | 发送时间戳 | 精确到毫秒 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| background_bottom_color | str | 底部背景色 | 十六进制颜色码 |
| background_color | str | 背景色 | 十六进制颜色码 |
| background_color_end | str | 背景渐变结束色 | 十六进制颜色码 |
| background_color_start | str | 背景渐变起始色 | 十六进制颜色码 |
| background_icon | str | 背景图标 | 图片URL，若无则为空字符串 |
| background_image | str | 背景图片 | 图片URL，若无则为空字符串 |
| background_price_color | str | 价格背景色 | 十六进制颜色码 |
| color_point | num | 待调查 | 例如0.7 |
| dmscore | num | 弹幕积分 | |
| end_time | num | 结束展示时间戳 | Unix时间戳，精确到秒 |
| gift | obj | 礼物信息 | 见下方展开 |
| group_medal | obj | 粉丝团勋章 | 见下方展开 |
| id | num | SC唯一ID | |
| is_mystery | bool | 是否神秘人 | |
| is_ranked | num | 是否进榜 | |
| is_send_audit | num | 是否发送审核 | |
| medal_info | obj | 佩戴的粉丝勋章信息 | 见下方展开 |
| message | str | 留言文本内容 | |
| message_font_color | str | 留言字体颜色 | 十六进制颜色码 |
| message_trans | str | 留言翻译内容 | 若无则为空字符串 |
| price | num | SC价格 | 单位：人民币（元） |
| rate | num | 兑换比例 | 通常为1000 |
| start_time | num | 开始展示时间戳 | Unix时间戳，精确到秒 |
| time | num | 展示时长 | 单位：秒 |
| token | str | 校验Token | |
| trans_mark | num | 翻译标记 | |
| ts | num | 发送时间戳 | Unix时间戳，精确到秒 |
| uid | num | 发送者UID | |
| uinfo | obj | 发送者详细信息 | 见下方展开 |
| user_info | obj | 发送者简要信息 | 见下方展开 |

data的gift字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_id | num | 礼物ID | 醒目留言通常固定为12000 |
| gift_name | str | 礼物名称 | 通常为"醒目留言" |
| num | num | 数量 | 通常为1 |

data的group_medal字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| is_lighted | num | 是否点亮 | |
| medal_id | num | 勋章ID | |
| name | str | 勋章名称 | |

data的medal_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| anchor_roomid | num | 主播直播间ID | |
| anchor_uname | str | 主播昵称 | |
| guard_level | num | 舰队等级 | 0无，1总督，2提督，3舰长 |
| icon_id | num | 图标ID | |
| is_lighted | num | 是否点亮 | 1点亮，0未点亮 |
| medal_color | str | 勋章颜色 | 十六进制颜色码 |
| medal_color_border | num | 边框颜色 | 十进制数值 |
| medal_color_end | num | 渐变结束色 | 十进制数值 |
| medal_color_start | num | 渐变起始色 | 十进制数值 |
| medal_level | num | 勋章等级 | |
| medal_name | str | 勋章名称 | |
| special | str | 特殊标识 | |
| target_id | num | 勋章归属者UID | |

data的user_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 头像链接 | |
| face_frame | str | 头像框链接 | |
| guard_level | num | 舰队等级 | |
| is_main_vip | num | 是否为主站VIP | |
| is_svip | num | 是否为超级VIP | |
| is_vip | num | 是否为VIP | |
| level_color | str | 等级颜色 | |
| manager | num | 是否房管 | 1是，0否 |
| name_color | str | 昵称颜色 | |
| title | str | 头衔 | |
| uname | str | 用户昵称 | |
| user_level | num | 用户等级 (UL) | |

data的uinfo字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| base | obj | 基础资料 | 见下方展开 |
| guard | obj | 舰队信息 | 见下方展开 |
| guard_leader | obj | 舰队队长状态 | 若无则为null |
| medal | obj | 粉丝勋章详情 | 见下方展开 |
| title | obj | 称号信息 | 见下方展开 |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| uid | num | 发送者UID | |
| wealth | obj | 财富信息 | 若无则为null |

data的uinfo的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 头像链接 | |
| is_mystery | bool | 是否神秘人 | |
| name | str | 用户昵称 | |
| name_color | num | 昵称颜色 | |
| name_color_str | str | 昵称颜色字符串 | |
| official_info | obj | 官方认证信息 | 见下方展开 |
| origin_info | obj | 原始信息 | 见下方展开 |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |

data的uinfo的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| desc | str | 认证描述 | |
| role | num | 认证角色 | |
| title | str | 认证头衔 | |
| type | num | 认证类型 | |

data的uinfo的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 原始头像链接 | |
| name | str | 原始昵称 | |

data的uinfo的guard字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| expired_str | str | 过期时间字符串 | |
| level | num | 舰队等级 | |

data的uinfo的medal字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| color | num | 颜色 | 十进制数值 |
| color_border | num | 边框颜色 | 十进制数值 |
| color_end | num | 渐变结束色 | 十进制数值 |
| color_start | num | 渐变起始色 | 十进制数值 |
| guard_icon | str | 舰队图标 | |
| guard_level | num | 舰队等级 | |
| honor_icon | str | 荣誉图标 | |
| id | num | 勋章ID | |
| is_light | num | 是否点亮 | 1点亮，0未点亮 |
| level | num | 勋章等级 | |
| name | str | 勋章名称 | |
| ruid | num | 勋章归属者UID | |
| score | num | 勋章积分 | |
| typ | num | 勋章类型 | |
| user_receive_count | num | 用户接收次数 | |
| v2_medal_color_border | str | V2边框颜色 | 十六进制颜色码带透明度 |
| v2_medal_color_end | str | V2渐变结束色 | |
| v2_medal_color_level | str | V2等级颜色 | |
| v2_medal_color_start | str | V2渐变起始色 | |
| v2_medal_color_text | str | V2文本颜色 | |

data的uinfo的title字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| old_title_css_id | str | 旧版称号CSS的ID | |
| title_css_id | str | 称号CSS的ID | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "SUPER_CHAT_MESSAGE",
  "data": {
    "background_bottom_color": "#2A60B2",
    "background_color": "#EDF5FF",
    "background_color_end": "#405D85",
    "background_color_start": "#3171D2",
    "background_icon": "",
    "background_image": "",
    "background_price_color": "#7497CD",
    "color_point": 0.7,
    "dmscore": 448,
    "end_time": 1775820797,
    "gift": {
      "gift_id": 12000,
      "gift_name": "醒目留言",
      "num": 1
    },
    "group_medal": {
      "is_lighted": 0,
      "medal_id": 0,
      "name": ""
    },
    "id": 15164370,
    "is_mystery": false,
    "is_ranked": 0,
    "is_send_audit": 0,
    "medal_info": {
      "anchor_roomid": 1964690546,
      "anchor_uname": "鹿野大王official",
      "guard_level": 0,
      "icon_id": 0,
      "is_lighted": 1,
      "medal_color": "#be6686",
      "medal_color_border": 12478086,
      "medal_color_end": 12478086,
      "medal_color_start": 12478086,
      "medal_level": 15,
      "medal_name": "鹿人A",
      "special": "",
      "target_id": 1396521412
    },
    "message": "点歌，勇气大爆发",
    "message_font_color": "#A3F6FF",
    "message_trans": "",
    "price": 30,
    "rate": 1000,
    "start_time": 1775820737,
    "time": 60,
    "token": "A305BEBA",
    "trans_mark": 0,
    "ts": 1775820737,
    "uid": 102581857,
    "uinfo": {
      "base": {
        "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
        "is_mystery": false,
        "name": "springtimes",
        "name_color": 0,
        "name_color_str": "#666666",
        "official_info": {
          "desc": "",
          "role": 0,
          "title": "",
          "type": -1
        },
        "origin_info": {
          "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
          "name": "springtimes"
        },
        "risk_ctrl_info": null
      },
      "guard": {
        "expired_str": "",
        "level": 0
      },
      "guard_leader": null,
      "medal": {
        "color": 12478086,
        "color_border": 12478086,
        "color_end": 12478086,
        "color_start": 12478086,
        "guard_icon": "",
        "guard_level": 0,
        "honor_icon": "",
        "id": 0,
        "is_light": 1,
        "level": 15,
        "name": "鹿人A",
        "ruid": 1396521412,
        "score": 559,
        "typ": 0,
        "user_receive_count": 0,
        "v2_medal_color_border": "#C770A499",
        "v2_medal_color_end": "#C770A499",
        "v2_medal_color_level": "#C770A4E6",
        "v2_medal_color_start": "#C770A499",
        "v2_medal_color_text": "#FFFFFF"
      },
      "title": {
        "old_title_css_id": "",
        "title_css_id": ""
      },
      "uhead_frame": null,
      "uid": 102581857,
      "wealth": null
    },
    "user_info": {
      "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
      "face_frame": "",
      "guard_level": 0,
      "is_main_vip": 1,
      "is_svip": 0,
      "is_vip": 0,
      "level_color": "#969696",
      "manager": 0,
      "name_color": "#666666",
      "title": "",
      "uname": "springtimes",
      "user_level": 7
    }
  },
  "is_report": true,
  "msg_id": "91505932605909505:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "send_time": 1775820737725
}
```
</details>

#### 醒目留言日文翻译

当收到醒目留言时，系统紧接着推送的包含日文机器翻译的该条留言信息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str  | "SUPER_CHAT_MESSAGE_JPN" | 如果是醒目留言日文版，内容则是"SUPER_CHAT_MESSAGE_JPN" |
| data | obj | SC详细信息 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| background_bottom_color | str | 底部背景色 | 十六进制颜色码 |
| background_color | str | 背景色 | 十六进制颜色码 |
| background_icon | str | 背景图标 | 图片URL，若无则为空字符串 |
| background_image | str | 背景图片 | 图片URL，若无则为空字符串 |
| background_price_color | str | 价格背景色 | 十六进制颜色码 |
| end_time | num | 结束展示时间戳 | Unix时间戳，精确到秒 |
| gift | obj | 礼物信息 | 见下方展开 |
| group_medal | obj | 粉丝团勋章 | 见下方展开 |
| id | num | SC唯一ID | 与原SC消息的ID一致 |
| is_mystery | bool | 是否神秘人 | |
| is_ranked | num | 是否进榜 | |
| medal_info | obj | 佩戴的粉丝勋章信息 | 见下方展开 |
| message | str | 原留言文本内容 | |
| message_jpn | str | 日文翻译留言 | 机器翻译出的日文内容 |
| price | num | SC价格 | 单位：人民币（元） |
| rate | num | 兑换比例 | 通常为1000 |
| start_time | num | 开始展示时间戳 | Unix时间戳，精确到秒 |
| time | num | 展示时长 | 单位：秒 |
| token | str | 校验Token | |
| ts | num | 发送时间戳 | Unix时间戳，精确到秒 |
| uid | num | 发送者UID | |
| uinfo | obj | 发送者详细信息 | 见下方展开 |
| user_info | obj | 发送者简要信息 | 见下方展开 |

data的gift字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_id | num | 礼物ID | 醒目留言通常固定为12000 |
| gift_name | str | 礼物名称 | 通常为"醒目留言" |
| num | num | 数量 | 通常为1 |

data的group_medal字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| is_lighted | num | 是否点亮 | |
| medal_id | num | 勋章ID | |
| name | str | 勋章名称 | |

data的medal_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| anchor_roomid | num | 主播直播间ID | |
| anchor_uname | str | 主播昵称 | |
| guard_level | num | 舰队等级 | 0无，1总督，2提督，3舰长 |
| icon_id | num | 图标ID | |
| is_lighted | num | 是否点亮 | 1点亮，0未点亮 |
| medal_color | str | 勋章颜色 | 十六进制颜色码 |
| medal_color_border | num | 边框颜色 | 十进制数值 |
| medal_color_end | num | 渐变结束色 | 十进制数值 |
| medal_color_start | num | 渐变起始色 | 十进制数值 |
| medal_level | num | 勋章等级 | |
| medal_name | str | 勋章名称 | |
| special | str | 特殊标识 | |
| target_id | num | 勋章归属者UID | |

data的user_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 头像链接 | |
| face_frame | str | 头像框链接 | |
| guard_level | num | 舰队等级 | |
| is_main_vip | num | 是否为主站VIP | |
| is_svip | num | 是否为超级VIP | |
| is_vip | num | 是否为VIP | |
| level_color | str | 等级颜色 | |
| manager | num | 是否房管 | 1是，0否 |
| name_color | str | 昵称颜色 | |
| title | str | 头衔 | |
| uname | str | 用户昵称 | |
| user_level | num | 用户等级 (UL) | |

data的uinfo字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| base | obj | 基础资料 | 见下方展开 |
| guard | obj | 舰队信息 | 见下方展开 |
| guard_leader | obj | 舰队队长状态 | 若无则为null |
| medal | obj | 粉丝勋章详情 | 见下方展开 |
| title | obj | 称号信息 | 见下方展开 |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| uid | num | 发送者UID | |
| wealth | obj | 财富信息 | 若无则为null |

data的uinfo的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 头像链接 | |
| is_mystery | bool | 是否神秘人 | |
| name | str | 用户昵称 | |
| name_color | num | 昵称颜色 | |
| name_color_str | str | 昵称颜色字符串 | |
| official_info | obj | 官方认证信息 | 见下方展开 |
| origin_info | obj | 原始信息 | 见下方展开 |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |

data的uinfo的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| desc | str | 认证描述 | |
| role | num | 认证角色 | |
| title | str | 认证头衔 | |
| type | num | 认证类型 | |

data的uinfo的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| face | str | 原始头像链接 | |
| name | str | 原始昵称 | |

data的uinfo的guard字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| expired_str | str | 过期时间字符串 | |
| level | num | 舰队等级 | |

data的uinfo的medal字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| color | num | 颜色 | 十进制数值 |
| color_border | num | 边框颜色 | 十进制数值 |
| color_end | num | 渐变结束色 | 十进制数值 |
| color_start | num | 渐变起始色 | 十进制数值 |
| guard_icon | str | 舰队图标 | |
| guard_level | num | 舰队等级 | |
| honor_icon | str | 荣誉图标 | |
| id | num | 勋章ID | |
| is_light | num | 是否点亮 | 1点亮，0未点亮 |
| level | num | 勋章等级 | |
| name | str | 勋章名称 | |
| ruid | num | 勋章归属者UID | |
| score | num | 勋章积分 | |
| typ | num | 勋章类型 | |
| user_receive_count | num | 用户接收次数 | |
| v2_medal_color_border | str | V2边框颜色 | 十六进制颜色码带透明度 |
| v2_medal_color_end | str | V2渐变结束色 | |
| v2_medal_color_level | str | V2等级颜色 | |
| v2_medal_color_start | str | V2渐变起始色 | |
| v2_medal_color_text | str | V2文本颜色 | |

data的uinfo的title字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| old_title_css_id | str | 旧版称号CSS的ID | |
| title_css_id | str | 称号CSS的ID | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "SUPER_CHAT_MESSAGE_JPN",
  "data": {
    "id": 15164370,
    "uid": 102581857,
    "price": 30,
    "rate": 1000,
    "message": "点歌，勇气大爆发",
    "message_jpn": "歌を注文して、勇気が爆発しました",
    "is_ranked": 0,
    "background_image": "",
    "background_color": "#EDF5FF",
    "background_icon": "",
    "background_price_color": "#7497CD",
    "background_bottom_color": "#2A60B2",
    "ts": 1775820738,
    "token": "B3522EB6",
    "medal_info": {
      "icon_id": 0,
      "target_id": 1396521412,
      "special": "",
      "anchor_uname": "鹿野大王official",
      "anchor_roomid": 1964690546,
      "medal_level": 15,
      "medal_name": "鹿人A",
      "medal_color": "#be6686",
      "medal_color_start": 12478086,
      "medal_color_end": 12478086,
      "medal_color_border": 12478086,
      "is_lighted": 1,
      "guard_level": 0
    },
    "user_info": {
      "uname": "springtimes",
      "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
      "face_frame": "",
      "name_color": "#666666",
      "guard_level": 0,
      "user_level": 7,
      "level_color": "#969696",
      "is_vip": 0,
      "is_svip": 0,
      "is_main_vip": 1,
      "title": "",
      "manager": 0
    },
    "time": 59,
    "start_time": 1775820737,
    "end_time": 1775820797,
    "gift": {
      "num": 1,
      "gift_id": 12000,
      "gift_name": "醒目留言"
    },
    "group_medal": {
      "medal_id": 0,
      "name": "",
      "is_lighted": 0
    },
    "is_mystery": false,
    "uinfo": {
      "uid": 102581857,
      "base": {
        "name": "springtimes",
        "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
        "name_color": 0,
        "is_mystery": false,
        "risk_ctrl_info": null,
        "origin_info": {
          "name": "springtimes",
          "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)"
        },
        "official_info": {
          "role": 0,
          "title": "",
          "desc": "",
          "type": -1
        },
        "name_color_str": "#666666"
      },
      "medal": {
        "name": "鹿人A",
        "level": 15,
        "color_start": 12478086,
        "color_end": 12478086,
        "color_border": 12478086,
        "color": 12478086,
        "id": 0,
        "typ": 0,
        "is_light": 1,
        "ruid": 1396521412,
        "guard_level": 0,
        "score": 559,
        "guard_icon": "",
        "honor_icon": "",
        "v2_medal_color_start": "#C770A499",
        "v2_medal_color_end": "#C770A499",
        "v2_medal_color_border": "#C770A499",
        "v2_medal_color_text": "#FFFFFF",
        "v2_medal_color_level": "#C770A4E6",
        "user_receive_count": 0
      },
      "wealth": null,
      "title": {
        "old_title_css_id": "",
        "title_css_id": ""
      },
      "guard": {
        "level": 0,
        "expired_str": ""
      },
      "uhead_frame": null,
      "guard_leader": null
    }
  }
}
```
</details>

#### 送礼

当用户在直播间投喂礼物（包括普通礼物、盲盒礼物爆出等）时触发。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "SEND_GIFT" | 投喂礼物时触发 |
| danmu | obj | 弹幕区域相关配置 | 例如 `{"area": 0}` |
| data | obj  | 礼物投喂人、礼物详情、盲盒信息、连击信息等 | 见下方展开 |
| msg_id | str | 消息唯一ID | 格式通常为时间戳与随机数的组合 |
| p_is_ack | bool | 待调查 | |
| p_msg_type | num | 待调查 | |
| send_time | num | 发送时间戳 | 精确到毫秒 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| action | str | 礼物操作 | 一般为"投喂" |
| bag_gift | obj/null | 背包礼物信息 | 若非背包礼物通常为null |
| batch_combo_id | str | 批量连击ID | 用于合并多次投喂的UI展示 |
| batch_combo_send | obj | 批量连击详情 | 见下方展开 |
| beatId | str | 待调查 | |
| benefits | obj/null | 待调查 | |
| biz_source | str | 业务来源 | 通常为"Live"或"live" |
| blind_gift | obj | 盲盒礼物信息 | **[重要字段]** 若为盲盒礼物则包含具体原始购买信息，普通礼物通常为null，见下方展开 |
| broadcast_id | num | 广播ID | |
| coin_type | str | 货币类型 | "gold" (金瓜子/电池) 或 "silver" (银瓜子) |
| combo_resources_id | num | 连击资源ID | |
| combo_send | obj | 连击详情信息 | 与batch_combo_send结构类似，见下方展开 |
| combo_stay_time | num | 连击UI停留时间 | 单位为秒 |
| combo_total_coin | num | 连击礼物总价值 | |
| crit_prob | num | 暴击概率 | |
| demarcation | num | 待调查 | |
| discount_price | num | 折扣价格 | |
| dmscore | num | 弹幕积分/展示权重 | 系统根据礼物价值打分，用于控制弹幕堆积时的渲染优先级 |
| draw | num | 待调查 | |
| effect | num | 待调查 | |
| effect_block | num | 特效屏蔽标志 | |
| effect_config | obj/null | 特效配置 | |
| face | str | 投喂者头像URL | |
| face_effect | obj | 待调查 | |
| face_effect_id | num | 待调查 | |
| face_effect_type | num | 待调查 | |
| face_effect_v2 | obj | 新版特效信息 | 包含 `id` 和 `type` |
| float_sc_resource_id | num | 待调查 | |
| giftId | num | 实际投喂/爆出的礼物ID | |
| giftName | str | 实际投喂/爆出的礼物名称 | |
| giftType | num | 礼物类型 | |
| gift_info | obj | 礼物视觉特效素材 | 见下方展开 |
| gift_tag | array | 礼物标签 | |
| gold | num | 待调查 | |
| group_medal | obj/null | 粉丝团勋章 | |
| guard_level | num | 投喂者的舰队等级 | 0无，1总督，2提督，3舰长 |
| is_first | bool | 是否首次送礼 | |
| is_join_receiver | bool | 待调查 | |
| is_naming | bool | 是否冠名 | |
| is_special_batch | num | 待调查 | |
| magnification | num | 抽奖倍率 | 盲盒或抽奖玩法中可能会出现倍数放大 |
| medal_info | obj | 投喂者粉丝勋章信息 | 结构同其他消息中的medal_info |
| name_color | str | 昵称颜色 | 十六进制颜色码 |
| num | num | 本次投喂的礼物数量 | |
| original_gift_name | str | 原始礼物名称 | 如果是盲盒，这里可能会有值 |
| price | num | 单个礼物价格 | |
| rcost | num | 待调查 | |
| receive_user_info | obj | 收礼人简要信息 | 包含 `uid` 和 `uname` |
| receiver_uinfo | obj | 收礼人详细信息 | 包含头像等结构化数据 (见下方 sender_uinfo 展开结构) |
| remain | num | 待调查 | |
| rnd | str | 随机特征码 | 通常用于礼物唯一标识防重 |
| send_master | obj/null | 待调查 | |
| sender_uinfo | obj | 投喂人详细信息 | 包含基础信息、勋章、舰队等，见下方展开 |
| silver | num | 银瓜子数 | |
| super | num | 待调查 | |
| super_batch_gift_num | num | 待调查 | |
| super_gift_num | num | 待调查 | |
| svga_block | num | 屏蔽SVGA特效标志 | |
| switch | bool | 待调查 | |
| tag_image | str | 标签图片URL | |
| tid | str | 交易ID / 时间戳特征码 | 与 rnd 类似 |
| timestamp | num | 送礼时间戳 | 精确到秒 |
| top_list | obj/null | 待调查 | |
| total_coin | num | 本次投喂总金额 | |
| uid | num | 投喂者UID | |
| uname | str | 投喂者昵称 | |
| wealth_level | num | 投喂者财富等级 | |

data的blind_gift字段 (盲盒信息)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| blind_gift_config_id | num | 盲盒配置ID | |
| from | num | 来源 | |
| gift_action | str | 礼物动作 | 例如"爆出"，表示从盲盒中开出的行为 |
| gift_tip_price | num | 提示价格 | 爆出礼物的实际价格 |
| original_gift_id | num | 原始购买的礼物ID | 盲盒本体的ID（如35206代表幸运盲盒） |
| original_gift_name | str | 原始购买的礼物名 | 例如"幸运盲盒" |
| original_gift_price | num | 原始礼物价格 | 盲盒购买时的价格 |

data的gift_info字段 (礼物视觉特效素材)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| effect_id | num | 特效ID | |
| gif | str | 动态图URL | 礼物的GIF动图链接 |
| has_imaged_gift | num | 是否包含图像 | |
| img_basic | str | 基础静态图URL | 礼物的PNG静态链接 |
| webp | str | WebP动画URL | 礼物的WebP动图链接 |

data的batch_combo_send / combo_send字段 (连击信息)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| action | str | 操作类型 | 一般为"投喂" |
| batch_combo_id / combo_id | str | 连击标识字符串 | |
| batch_combo_num / combo_num | num | 连击累计数量 | |
| blind_gift | obj | 盲盒信息 | 结构同上方的 blind_gift |
| gift_id | num | 礼物ID | |
| gift_name | str | 礼物名称 | |
| gift_num | num | 单次增加数量 | |
| send_master | obj/null | 待调查 | |
| uid | num | 投喂者UID | |
| uname | str | 投喂者昵称 | |

data的sender_uinfo字段 (投喂人详细信息)
*(注：receiver_uinfo 结构与之完全相同)*

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| uid | num | 投喂者UID | |
| base | obj | 基础资料 | 包含 name, face, is_mystery 等 |
| medal | obj | 勋章详情 | 若无则为null |
| wealth | obj | 财富信息 | 若无则为null |
| title | obj | 称号信息 | 若无则为null |
| guard | obj | 舰队信息 | 若无则为null |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| guard_leader | obj | 舰队队长状态 | 若无则为null |


<details>
<summary>查看消息示例 (带盲盒机制的新版)：</summary>

```json
{
  "cmd": "SEND_GIFT",
  "danmu": {
    "area": 0
  },
  "data": {
    "action": "投喂",
    "bag_gift": null,
    "batch_combo_id": "490a8c58-bc70-4b96-b266-99a8d1429cbc",
    "batch_combo_send": {
      "action": "投喂",
      "batch_combo_id": "490a8c58-bc70-4b96-b266-99a8d1429cbc",
      "batch_combo_num": 1,
      "blind_gift": {
        "blind_gift_config_id": 144,
        "from": 0,
        "gift_action": "爆出",
        "gift_tip_price": 2500,
        "original_gift_id": 35206,
        "original_gift_name": "幸运盲盒",
        "original_gift_price": 5000
      },
      "gift_id": 35311,
      "gift_name": "好运柚叶",
      "gift_num": 1,
      "send_master": null,
      "uid": 35623247,
      "uname": "不是Kennya"
    },
    "beatId": "0",
    "benefits": null,
    "biz_source": "Live",
    "blind_gift": {
      "blind_gift_config_id": 144,
      "from": 0,
      "gift_action": "爆出",
      "gift_tip_price": 2500,
      "original_gift_id": 35206,
      "original_gift_name": "幸运盲盒",
      "original_gift_price": 5000
    },
    "broadcast_id": 0,
    "coin_type": "gold",
    "combo_resources_id": 1,
    "combo_send": {
      "action": "投喂",
      "combo_id": "f3609c1b-d665-432f-b512-ca574f6f3c5e",
      "combo_num": 1,
      "gift_id": 35311,
      "gift_name": "好运柚叶",
      "gift_num": 1,
      "send_master": null,
      "uid": 35623247,
      "uname": "不是Kennya"
    },
    "combo_stay_time": 5,
    "combo_total_coin": 2500,
    "crit_prob": 0,
    "demarcation": 2,
    "discount_price": 2500,
    "dmscore": 714,
    "draw": 0,
    "effect": 0,
    "effect_block": 0,
    "effect_config": null,
    "face": "[https://i2.hdslb.com/bfs/face/c21d3e894d4451bea6339c8563bcd64133447d80.jpg](https://i2.hdslb.com/bfs/face/c21d3e894d4451bea6339c8563bcd64133447d80.jpg)",
    "face_effect": {},
    "face_effect_id": 0,
    "face_effect_type": 0,
    "face_effect_v2": {
      "id": 0,
      "type": 0
    },
    "float_sc_resource_id": 0,
    "giftId": 35311,
    "giftName": "好运柚叶",
    "giftType": 0,
    "gift_info": {
      "effect_id": 0,
      "gif": "[https://i0.hdslb.com/bfs/live/5396725f058789056be3da27c66a570d1500c512.gif](https://i0.hdslb.com/bfs/live/5396725f058789056be3da27c66a570d1500c512.gif)",
      "has_imaged_gift": 0,
      "img_basic": "[https://s1.hdslb.com/bfs/live/792e1ae4e92cc33fa323970c55464ff6240959f0.png](https://s1.hdslb.com/bfs/live/792e1ae4e92cc33fa323970c55464ff6240959f0.png)",
      "webp": "[https://i0.hdslb.com/bfs/live/0aa76c0b4ab72cc93a7e28012088fac1f53baf0d.webp](https://i0.hdslb.com/bfs/live/0aa76c0b4ab72cc93a7e28012088fac1f53baf0d.webp)"
    },
    "gift_tag": [],
    "gold": 0,
    "group_medal": null,
    "guard_level": 3,
    "is_first": true,
    "is_join_receiver": false,
    "is_naming": false,
    "is_special_batch": 0,
    "magnification": 1,
    "medal_info": {
      "anchor_roomid": 0,
      "anchor_uname": "",
      "guard_level": 3,
      "icon_id": 0,
      "is_lighted": 1,
      "medal_color": 398668,
      "medal_color_border": 6809855,
      "medal_color_end": 6850801,
      "medal_color_start": 398668,
      "medal_level": 27,
      "medal_name": "莓哈哟",
      "special": "",
      "target_id": 3546786748697162
    },
    "name_color": "#00D1F1",
    "num": 1,
    "original_gift_name": "",
    "price": 2500,
    "rcost": 89007,
    "receive_user_info": {
      "uid": 3706947196946570,
      "uname": "早早眠-"
    },
    "receiver_uinfo": {
      "base": {
        "face": "[https://i2.hdslb.com/bfs/face/5579b93ac70dd2c2e1d33217aa747b0090dbb07d.jpg](https://i2.hdslb.com/bfs/face/5579b93ac70dd2c2e1d33217aa747b0090dbb07d.jpg)",
        "name": "早早眠-",
        "name_color_str": ""
      },
      "uid": 3706947196946570
    },
    "remain": 0,
    "rnd": "4760043420746102272",
    "send_master": null,
    "sender_uinfo": {
      "base": {
        "face": "[https://i2.hdslb.com/bfs/face/c21d3e894d4451bea6339c8563bcd64133447d80.jpg](https://i2.hdslb.com/bfs/face/c21d3e894d4451bea6339c8563bcd64133447d80.jpg)",
        "name": "不是Kennya"
      },
      "medal": {
        "guard_level": 3,
        "level": 26,
        "name": "早早味"
      },
      "uid": 35623247
    },
    "silver": 0,
    "super": 0,
    "super_batch_gift_num": 1,
    "super_gift_num": 1,
    "svga_block": 0,
    "switch": true,
    "tag_image": "",
    "tid": "4760043420746102272",
    "timestamp": 1775827219,
    "top_list": null,
    "total_coin": 5000,
    "uid": 35623247,
    "uname": "不是Kennya",
    "wealth_level": 24
  },
  "msg_id": "91512729688348672:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "send_time": 1775827219928
}
```
</details>

#### 礼物星球进度变动

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str | "WIDGET_GIFT_STAR_PROCESS" | 礼物星球进度变动事件。 |
| data | obj | 挂件任务详情 | 包含任务起止时间及所需收集的礼物进度列表。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| start_date | num | 任务开始日期 | 格式为 YYYYMMDD（如 `20260330`）。 |
| process_list | array | 收集进度列表 | **核心数据，包含部分需要收集的礼物及其进度。** |
| finished | bool | 是否已完成 | 当前活动任务是否全部达标。 |
| ddl_timestamp | num | 截止时间戳 | 任务结束的 Unix 时间戳（秒）。 |
| version | num | 版本戳 | 毫秒级时间戳。 |
| reward_gift | num | 奖励礼物ID | 完成任务后触发或奖励的礼物 ID。 |
| reward_gift_img | str | 奖励礼物图片 | |
| reward_gift_name | str | 奖励礼物名称 | |
| level_info | obj/null | 等级信息 | 待调查。 |

data.process_list数组 (进度列表)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| gift_id | num | 需要收集的礼物ID | |
| gift_img | str | 礼物图片URL | |
| gift_name | str | 礼物名称 | 不知道为什么全都是“礼物星球”，待调查。 |
| completed_num | num | 已完成数量 | 当前已收集到该礼物的数量。 |
| target_num | num | 目标收集数量 | 该项礼物需要收集的总数。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "WIDGET_GIFT_STAR_PROCESS",
  "data": {
    "start_date": 20260330,
    "process_list": [
      {
        "gift_id": 35432,
        "gift_img": "https://s1.hdslb.com/bfs/live/d927e689c3f33642e5283fc44469794eef017d94.png",
        "gift_name": "礼物星球",
        "completed_num": 0,
        "target_num": 10
      },
      {
        "gift_id": 35520,
        "gift_img": "https://s1.hdslb.com/bfs/live/8c883690aa927c629da461f793bcf01c98143d0d.png",
        "gift_name": "礼物星球",
        "completed_num": 0,
        "target_num": 10
      },
      {
        "gift_id": 35502,
        "gift_img": "https://s1.hdslb.com/bfs/live/f307cb4f29be9f13b6610a8ebbdd8acffa068ba9.png",
        "gift_name": "礼物星球",
        "completed_num": 0,
        "target_num": 1
      }
    ],
    "finished": false,
    "ddl_timestamp": 1775404800,
    "version": 1774805204830,
    "reward_gift": 0,
    "reward_gift_img": "",
    "reward_gift_name": "",
    "level_info": null
  }
}
```
</details>

#### 通用系统广播弹幕

B站的通用系统通知弹幕。当观众触发特定高光成就（如点亮礼物星球、开通特殊守护等）时下发。服务器通常会为适配不同终端的 UI（如浅色/深色模式、Web/App端），下发多条带有不同 `terminals` 和颜色配置的同类消息。

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "COMMON_NOTICE_DANMAKU" | 通用系统通知弹幕。 |
| data | obj | 通知详情 | 包含富文本分段和目标终端数组。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| content_segments | array | 富文本内容段 | **核心数据，包含具体的广播文本和颜色配置。** |
| dmscore | num | 互动积分 | |
| terminals | array | 目标终端编号 | 标识该配置适用于哪些客户端。例如 `[1, 2, 3]` 通常为 Web 端，`[4, 5]` 通常为移动端。 |

data.content_segments数组 (富文本内容段)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| font_color | str | 默认字体颜色 | Hex 颜色值（如 `"#FFFFFF"`）。 |
| font_color_dark | str | 深色模式字体颜色 | Hex 颜色值（如 `"#a2a7ae"`）。 |
| highlight_font_color | str | 高亮字体颜色 | Hex 颜色值（如 `"#FFB027"`）。 |
| highlight_font_color_dark | str | 深色模式高亮颜色 | Hex 颜色值。 |
| text | str | 广播文本内容 | **需要高亮的词汇通常包裹在 `<%` 和 `%>` 中**。例如 `"<%小花花%> 被点亮啦！恭喜 <%mikufilck%> 成为星球守护者！"`。 |
| type | num | 文本类型 | 例如 `1`。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "COMMON_NOTICE_DANMAKU",
  "data": {
    "content_segments": [
      {
        "font_color": "#FFFFFF",
        "font_color_dark": "#FFFFFF",
        "highlight_font_color": "#FFB027",
        "highlight_font_color_dark": "#FFB027",
        "text": "<%小花花%> 被点亮啦！恭喜 <%mikufilck%> 成为星球守护者！",
        "type": 1
      }
    ],
    "dmscore": 1008,
    "terminals": [1, 2, 3]
  }
}
```
</details>

#### 礼物连击

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str | "COMBO_SEND" | 触发礼物连击特效。 |
| data | obj | 连击详情 | 包含送礼人、收礼人以及连击数量的精确数据。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| action | str | 动作文本 | 通常为 `"投喂"` 或 `"赠送"`。 |
| batch_combo_id | str | 批量连击批次ID | 用于追踪这一次连续点击行为。 |
| batch_combo_num | num | 批次内连击数量 | |
| coin_type | str | 货币类型 | 如 `"gold"` (金瓜子) 或 `"silver"` (银瓜子)。 |
| combo_id | str | 连击唯一ID | 格式如 `"gift:combo_id:送礼人UID:收礼人UID:礼物ID:时间戳"`。 |
| combo_num | num | **当前连击数** | 例如 `5` 代表目前是 5 连击，**前端直接用此值渲染 x5 效果**。 |
| combo_total_coin | num | 连击累计消费 | 本次连击累计消耗的瓜子总数。 |
| dmscore | num | 互动积分 | |
| gift_id | num | 礼物ID | 唯一标识是哪种礼物（如 `31036` 代表小花花）。 |
| gift_info | obj | 礼物图像信息 | 包含礼物的图片 URL。 |
| gift_name | str | 礼物名称 | |
| gift_num | num | 单次礼物数量 | 在 COMBO 包中通常为 `0` 或单次点击的数量。计算总数应看 `total_num` 或 `combo_num`。 |
| group_medal | obj/null | 粉丝团信息 | 待调查，通常为 null。 |
| is_join_receiver | bool | 是否加入收礼人 | 待调查。 |
| is_naming | bool | 是否冠名 | 待调查。 |
| is_show | num | 是否显示 | `1` 为显示。 |
| medal_info | obj | **旧版粉丝牌信息** | **送礼人当前佩戴的粉丝勋章详情（扁平结构）。** |
| name_color | str | 名字颜色 | |
| ruid | num | 收礼人UID | |
| r_uname | str | 收礼人昵称 | 多人连麦时，可用来判断礼物送给了哪个主播。 |
| receive_user_info | obj | 简易收礼人信息 | 包含基础的 UID 和昵称。 |
| receiver_uinfo | obj | 收礼人底层信息 | 包含收礼主播的精确基础数据（新版结构）。 |
| send_master | obj/null | 赠送控制者 | 待调查。 |
| sender_uinfo | obj | 送礼人底层信息 | 包含送礼人基础数据及新版粉丝牌结构。 |
| total_num | num | 连击总数量 | 与 `combo_num` 类似。 |
| uid | num | 送礼人UID | |
| uname | str | 送礼人昵称 | |
| wealth_level | num | 财富等级 | 送礼人的全站财富等级。 |

data.gift_info字段 (礼物图片)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| img_basic | str | 基础静态图片 | |
| webp | str | 动态/高清 WebP 图片 | |

data.medal_info字段 (送礼人旧版粉丝勋章)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| target_id | num | 目标UID | 发放该粉丝牌的主播 UID。 |
| medal_name | str | 勋章名称 | |
| medal_level | num | 勋章等级 | |
| medal_color | num | 基础颜色 | 十进制颜色值。 |
| medal_color_start | num | 渐变起始色 | 十进制颜色值。 |
| medal_color_end | num | 渐变结束色 | 十进制颜色值。 |
| medal_color_border | num | 边框颜色 | 十进制颜色值。 |
| is_lighted | num | 是否点亮 | `1`点亮，`0`熄灭。 |
| guard_level | num | 大航海等级 | `0`无，`1`总督，`2`提督，`3`舰长。 |
| icon_id | num | 图标ID | |
| anchor_roomid | num | 主播房间号 | 发放该粉丝牌的直播间号（通常为 0，需调查）。 |
| anchor_uname | str | 主播昵称 | |
| special | str | 特殊标识 | |

data.receive_user_info字段 (简易收礼人信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 收礼人UID | |
| uname | str | 收礼人昵称 | |

data.sender_uinfo字段 (送礼人新版底层信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| base | obj | 基础信息 | 包含 `name` 和 `face` 等精确基础数据。 |
| medal | obj | 粉丝勋章信息 | 包含送礼人当前佩戴的粉丝牌详情 (推荐使用此新版结构获取 V2 颜色字符串)。 |
| guard | obj/null | 舰队信息 | |

data.receiver_uinfo字段 (收礼人新版底层信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| base | obj | 基础信息 | 包含 `name` 和 `face` 等精确基础数据。 |
| uid | num | 收礼人UID | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "COMBO_SEND",
  "data": {
    "action": "投喂",
    "batch_combo_id": "batch:gift:combo_id:26928797:3546607121336514:31036:1774804255.6305",
    "batch_combo_num": 5,
    "coin_type": "gold",
    "combo_id": "gift:combo_id:26928797:3546607121336514:31036:1774804255.6294",
    "combo_num": 5,
    "combo_total_coin": 500,
    "dmscore": 896,
    "gift_id": 31036,
    "gift_info": {
      "img_basic": "[https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png](https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png)",
      "webp": "[https://i0.hdslb.com/bfs/live/28357ba4cd566418730ca29da2c552efa7e4a390.webp](https://i0.hdslb.com/bfs/live/28357ba4cd566418730ca29da2c552efa7e4a390.webp)"
    },
    "gift_name": "小花花",
    "gift_num": 0,
    "group_medal": null,
    "is_join_receiver": false,
    "is_naming": false,
    "is_show": 1,
    "medal_info": {
      "anchor_roomid": 0,
      "anchor_uname": "",
      "guard_level": 0,
      "icon_id": 0,
      "is_lighted": 0,
      "medal_color": 2951253,
      "medal_color_border": 12632256,
      "medal_color_end": 12632256,
      "medal_color_start": 12632256,
      "medal_level": 29,
      "medal_name": "挽抑云",
      "special": "",
      "target_id": 1899012758
    },
    "name_color": "",
    "r_uname": "枳念念",
    "receive_user_info": {
      "uid": 3546607121336514,
      "uname": "枳念念"
    },
    "receiver_uinfo": {
      "base": {
        "face": "[https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg](https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg)",
        "is_mystery": false,
        "name": "枳念念",
        "name_color": 0,
        "name_color_str": "",
        "official_info": { "desc": "", "role": 0, "title": "", "type": -1 },
        "origin_info": { "face": "[https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg](https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg)", "name": "枳念念" },
        "risk_ctrl_info": null
      },
      "guard": null, "guard_leader": null, "medal": null, "title": null, "uhead_frame": null,
      "uid": 3546607121336514, "wealth": null
    },
    "ruid": 3546607121336514,
    "send_master": null,
    "sender_uinfo": {
      "base": {
        "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
        "is_mystery": false,
        "name": "mikufilck",
        "name_color": 0,
        "name_color_str": "",
        "official_info": { "desc": "", "role": 0, "title": "", "type": -1 },
        "origin_info": { "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)", "name": "mikufilck" },
        "risk_ctrl_info": null
      },
      "guard": null, "guard_leader": null,
      "medal": {
        "color": 13081892, "color_border": 13081892, "color_end": 13081892, "color_start": 13081892,
        "guard_icon": "", "guard_level": 0, "honor_icon": "", "id": 0, "is_light": 1, "level": 19,
        "name": "枳上语", "ruid": 3546607121336514, "score": 1433, "typ": 0, "user_receive_count": 0,
        "v2_medal_color_border": "#C770A499", "v2_medal_color_end": "#C770A499",
        "v2_medal_color_level": "#C770A4E6", "v2_medal_color_start": "#C770A499", "v2_medal_color_text": "#FFFFFF"
      },
      "title": null, "uhead_frame": null, "uid": 26928797, "wealth": null
    },
    "total_num": 5,
    "uid": 26928797,
    "uname": "mikufilck",
    "wealth_level": 32
  }
}
```
</details>

<!--
#### 欢迎加入房间

#### 欢迎房管加入房间
-->

#### 通知消息

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "NOTICE_MSG" | 通知消息，内容则是"NOTICE_MSG" |
| id | num | 待调查 | |
| full | obj | 待调查 | |
| half | obj | 待调查 | |
| side | obj | 待调查 | |
| roomid | num | 目标直播间短号 | |
| real_roomid | num | 目标直播间真实ID | |
| msg_common | str | 显示的消息内容 | |
| msg_self | str | 消息内容本身 | 剔除额外文本 |
| link_rel | str | 通知消息跳转的URL | |
| msg_type | num | 待调查 | |
| shield_uid | num | 待调查 | |
| business_id | str | 待调查 | |
| scatter | obj | 待调查 | |
| marquee_id | str | 待调查 | |
| notice_type | num | 待调查 | |

<details>
<summary>查看消息示例：</summary>

```json
{
    "cmd": "NOTICE_MSG",
    "id": 804,
    "name": "人气榜第一名",
    "full": {
        "head_icon": "https://i0.hdslb.com/bfs/live/f74b09c7fb83123a0dd66c536b6d5b143d271b08.png",
        "tail_icon": "https://i0.hdslb.com/bfs/live/822da481fdaba986d738db5d8fd469ffa95a8fa1.webp",
        "head_icon_fa": "https://i0.hdslb.com/bfs/live/f74b09c7fb83123a0dd66c536b6d5b143d271b08.png",
        "tail_icon_fa": "https://i0.hdslb.com/bfs/live/38cb2a9f1209b16c0f15162b0b553e3b28d9f16f.png",
        "head_icon_fan": 1,
        "tail_icon_fan": 4,
        "background": "#FFE6BD",
        "color": "#9D5412",
        "highlight": "#FF6933",
        "time": 20
    },
    "half": {
        "head_icon": "https://i0.hdslb.com/bfs/live/f74b09c7fb83123a0dd66c536b6d5b143d271b08.png",
        "tail_icon": "https://i0.hdslb.com/bfs/live/822da481fdaba986d738db5d8fd469ffa95a8fa1.webp",
        "background": "#FFE6BD",
        "color": "#9D5412",
        "highlight": "#FF6933",
        "time": 0
    },
    "side": {
        "head_icon": "",
        "background": "",
        "color": "",
        "highlight": "",
        "border": ""
    },
    "roomid": 23919301,
    "real_roomid": 23919301,
    "msg_common": "恭喜主播<%AG超玩会王者荣耀一诺%>荣获上小时人气榜第<%1%>名！点击传送查看精彩内容！",
    "msg_self": "恭喜主播<%AG超玩会王者荣耀一诺%>荣获上小时人气榜第<%1%>名！",
    "link_url": "https://live.bilibili.com/23919301?broadcast_type=0&is_room_feed=1&from=28003&extra_jump_from=28003",
    "msg_type": 1,
    "shield_uid": -1,
    "business_id": "",
    "scatter": {
        "min": 0,
        "max": 0
    },
    "marquee_id": "",
    "notice_type": 0
}
```
```json
{
    "cmd": "NOTICE_MSG",
    "id": 814,
    "name": "幻影飞船专用",
    "full": {
        "head_icon": "https://i0.hdslb.com/bfs/live/08978f1721200e11328d1f7d6231b21bcca20488.gif",
        "tail_icon": "https://i0.hdslb.com/bfs/live/822da481fdaba986d738db5d8fd469ffa95a8fa1.webp",
        "head_icon_fa": "https://i0.hdslb.com/bfs/live/08978f1721200e11328d1f7d6231b21bcca20488.gif",
        "tail_icon_fa": "https://i0.hdslb.com/bfs/live/38cb2a9f1209b16c0f15162b0b553e3b28d9f16f.png",
        "head_icon_fan": 1,
        "tail_icon_fan": 4,
        "background": "#F09153",
        "color": "#FFFFFF",
        "highlight": "#FFE600",
        "time": 15
    },
    "half": {
        "head_icon": "https://i0.hdslb.com/bfs/live/08978f1721200e11328d1f7d6231b21bcca20488.gif",
        "tail_icon": "",
        "background": "#F09153",
        "color": "#FFFFFFFF",
        "highlight": "#FFE600",
        "time": 15
    },
    "side": {
        "head_icon": "",
        "background": "",
        "color": "",
        "highlight": "",
        "border": ""
    },
    "roomid": 25207004,
    "real_roomid": 25207004,
    "msg_common": "<%咖啡_ミシェル%>投喂<%夜月瓜瓜sukuyi%>1个幻影飞船，向着浩瀚星辰出发！",
    "msg_self": "<%咖啡_ミシェル%>投喂<%夜月瓜瓜sukuyi%>1个幻影飞船，向着浩瀚星辰出发！",
    "link_url": "https://live.bilibili.com/25207004?broadcast_type=0&is_room_feed=1&from=28003&extra_jump_from=28003&live_lottery_type=1",
    "msg_type": 2,
    "shield_uid": -1,
    "business_id": "32356",
    "scatter": {
        "min": 0,
        "max": 0
    },
    "marquee_id": "",
    "notice_type": 0
}
```

</details>

#### 主播准备中

当主播结束直播（下播）或直播间进入“准备中”状态时接收到此消息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "PREPARING" | 指示直播间进入准备中/下播状态 |
| roomid | str | 直播间ID | 注意：实际数据中为字符串格式。未知是真实ID还是短号，但通常为真实ID |
| round | num | 轮播状态 | **[旧版/可选字段]** 1正在轮播，0未轮播。新版数据中观察到该字段可能被省略 |
| msg_id | str | 消息唯一ID | **[新版新增]** 格式通常为时间戳与随机数的组合 |
| p_is_ack | bool | 待调查 | **[新版新增]** |
| p_msg_type | num | 待调查 | **[新版新增]** |
| send_time | num | 发送时间戳 | **[新版新增]** 精确到毫秒 |

<details>
<summary>查看消息示例 (新版)：</summary>

```json
{
  "cmd": "PREPARING",
  "msg_id": "91518955142857216:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "roomid": "30827248",
  "send_time": 1775833156984
}
```

</details>

#### 直播开始

当主播开始直播（开播）时接收到此消息。与 `PREPARING`（准备中/下播）指令相对应。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "LIVE" | 指示直播间正式进入开播状态 |
| live_key | str | 直播场次密钥 | 该场直播的全局唯一标识符 |
| voice_background | str | 语音背景图 | 音频/电台直播时的背景图URL，视频直播通常为空字符串 |
| sub_session_key | str | 子场次密钥 | 场次标识，通常由 `live_key` 和开播时间戳（sub_time）拼接而成 |
| live_platform | str | 开播平台 | 主播使用的推流工具/平台标识（例如 `"pc_link"` 代表PC端推流） |
| live_model | num | 直播模式 | 待调查（例如0可能代表常规视频直播） |
| roomid | num | 直播间ID | **【注意】** 在此指令中为数字类型（而在 `PREPARING` 指令中常为字符串格式） |
| live_time | num | 开播时间戳 | 服务器记录的正式开播 Unix 时间戳，精确到秒 |
| special_types | array | 特殊类型列表 | 待调查（可能用于标记特定活动的特殊直播间，通常为空数组） |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "LIVE",
  "live_key": "693870340542020839",
  "voice_background": "",
  "sub_session_key": "693870340542020839sub_time:1775986526",
  "live_platform": "pc_link",
  "live_model": 0,
  "roomid": 3128551,
  "live_time": 1775986526,
  "special_types": []
}
```
</details>


#### 主播信息更新

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "ROOM_REAL_TIME_MESSAGE_UPDATE" | |
| data | obj | 房间ID、主播粉丝数等 | |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| roomid | num | 直播间ID | 未知是真实ID还是短号 | |
| fans | num | 主播当前粉丝数 | |
| red_notice | num | 待调查 | |
| fans_club | num | 主播粉丝团人数 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "ROOM_REAL_TIME_MESSAGE_UPDATE",
    "data": {
        "roomid": 8618057,
        "fans": 136,
        "red_notice": -1,
        "fans_club": 8
    }
}
```
</details>

#### 直播间高能榜

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "ONLINE_RANK_V2" | 直播间高能用户数据刷新，内容则是"ONLINE_RANK_V2" |
| data | obj | 直播间高能用户数据 | |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| list | array | 在直播间高能用户中的用户信息 | |
| rank_type | str | 待调查 | |

list数组中的对象

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| uid | num | 用户UID | |
| face | str | 用户头像URL | |
| score | str | 该用户的贡献值 | |
| uname | str | 用户名称 | |
| rank | num | 该用户在高能榜中的排名 | |
| guard_level | num | 待调查 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "ONLINE_RANK_V2",
    "data": {
        "list": [
            {
                "uid": 2082621455,
                "face": "https://i2.hdslb.com/bfs/face/9de6050277fa13d830eb97e3453d89843de46a31.jpg",
                "score": "20",
                "uname": "8级萌新_小华",
                "rank": 1,
                "guard_level": 0
            },
            {
                "uid": 50500335,
                "face": "https://i0.hdslb.com/bfs/face/ca722209251478ef0ffb45c3adeafb9dab283c57.jpg",
                "score": "20",
                "uname": "属官一号",
                "rank": 2,
                "guard_level": 0
            },
            {
                "uid": 29857468,
                "face": "https://i1.hdslb.com/bfs/face/7b4ae2e7e950f2dfb2bd969859c813487ce3b64c.jpg",
                "score": "12",
                "uname": "露萌不要雨草",
                "rank": 3,
                "guard_level": 0
            }
        ],
        "rank_type": "gold-rank"
    }
}
```
  
</details>


#### 直播间高能用户数量

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "ONLINE_RANK_COUNT" | 直播间高能用户数，内容是"ONLINE_RANK_COUNT" |
| data | obj | 直播间高能用户数量 | |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| count | num | 直播间高能用户数量 | |


<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "ONLINE_RANK_COUNT",
    "data": {
        "count": 4
    }
}
```
  
</details>

#### 用户到达直播间高能榜前三名的消息


json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "ONLINE_RANK_TOP3" | |
| data | obj | 消息内容、高能榜排名等 | |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| dmscore | num | 待调查 | |
| list | array | 消息内容和高能榜排名 | |

list数组中的对象

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| msg | str | 消息内容 | |
| rank | num | 该用户的高能榜排名 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "ONLINE_RANK_TOP3",
    "data": {
        "dmscore": 112,
        "list": [
            {
                "msg": "恭喜 <%你干嘛哈哈哎哟%> 成为高能用户",
                "rank": 1
            }
        ]
    }
}
```

</details>

#### 主播排行榜变动

当主播在直播间的各大排行榜（如热门榜、小时榜、高能榜等）上的名次发生变动、或者上榜/下榜时接收到此消息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "RANK_CHANGED" | 指示主播的打榜排名发生变动 |
| data | obj  | 排行榜详细变动信息 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| uid | num | 主播UID | 当前直播间主播的UID |
| rank | num | 综合排名 | 当前的综合排名（0通常表示未上榜或名次靠后未显示具体的数字） |
| countdown | num | 倒计时 | 排行榜结算的倒计时（秒） |
| timestamp | num | 时间戳 | 数据包发送的Unix时间戳，精确到秒 |
| url | str | 通用榜单链接 | 若无则为空字符串 |
| on_rank_name_by_type | str | 上榜名称 | 例如"热门榜" |
| rank_name_by_type | str | 分类榜单名称 | 例如"热门榜" |
| url_by_type | str | 分类榜单H5链接 | 包含榜单具体UI参数和主播ID参数的网页链接 |
| rank_by_type | num | 分类排名 | 主播在该具体分类榜单中的名次（0通常表示未进入显示范围） |
| rank_type | num | 排行榜类型 | 排行榜的内部类型枚举值（例如3代表某种特定的热门榜） |
| sub_rank_type | num | 子排行榜类型 | 该榜单下的子分类枚举值 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "RANK_CHANGED",
  "data": {
    "uid": 3546384651258599,
    "rank": 0,
    "countdown": 0,
    "timestamp": 1775833156,
    "url": "",
    "on_rank_name_by_type": "热门榜",
    "rank_name_by_type": "热门榜",
    "url_by_type": "[https://live.bilibili.com/p/html/live-app-hotrank/index.html?is_live_half_webview=1&hybrid_rotate_d=1&hybrid_half_ui=1,3,100p,70p,0,0,30,100,12;2,2,375,100p,0,0,30,100,0;3,3,100p,70p,0,0,30,100,12;4,2,375,100p,0,0,30,100,0;5,3,100p,70p,0,0,30,100,0;6,3,100p,70p,0,0,30,100,0;7,3,100p,70p,0,0,30,100,0;8,3,100p,70p,0,0,30,100,0&pc_ui=338,465,f4eefa,0&redirect=v2&rank=hot&anchorId=3546384651258599&rank_type=1](https://live.bilibili.com/p/html/live-app-hotrank/index.html?is_live_half_webview=1&hybrid_rotate_d=1&hybrid_half_ui=1,3,100p,70p,0,0,30,100,12;2,2,375,100p,0,0,30,100,0;3,3,100p,70p,0,0,30,100,12;4,2,375,100p,0,0,30,100,0;5,3,100p,70p,0,0,30,100,0;6,3,100p,70p,0,0,30,100,0;7,3,100p,70p,0,0,30,100,0;8,3,100p,70p,0,0,30,100,0&pc_ui=338,465,f4eefa,0&redirect=v2&rank=hot&anchorId=3546384651258599&rank_type=1)",
    "rank_by_type": 0,
    "rank_type": 3,
    "sub_rank_type": 0
  }
}
```

</details>

#### 榜单排名刷新

当主播切换直播分区，或者涉及到特定排行榜模块发生重大变动时，系统会下发此指令，强制要求客户端重新拉取并刷新对应的排行榜数据。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "CHG_RANK_REFRESH" | 指示客户端刷新排行榜数据 |
| data | obj  | 刷新榜单的具体参数 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| cmd | str | 内部命令 | 再次嵌套了 "CHG_RANK_REFRESH" |
| rank_type | num | 排行榜类型 | 排行榜的内部类型枚举值（如 3 对应特定的榜单类型） |
| rank_module | str | 排行榜所属模块 | 触发刷新的模块。例如 `"area"` 代表**分区榜单**（通常在主播切换分区后触发） |
| room_id | num | 直播间ID | 当前直播间的数字 ID |
| ruid | num | 主播UID | 当前直播间主播的 UID |
| need_refresh | bool | 是否需要刷新 | `true` 代表要求前端立刻执行列表刷新动作 |
| version | num | 版本号/时间戳 | 格式通常为毫秒级时间戳（如 1775987047867），用于前端对比数据的新旧版本以防止冲突 |

<details>
<summary>查看消息示例 (因切换分区触发的分区榜刷新)：</summary>

```json
{
  "cmd": "CHG_RANK_REFRESH",
  "data": {
    "cmd": "CHG_RANK_REFRESH",
    "rank_type": 3,
    "rank_module": "area",
    "room_id": 3128551,
    "ruid": 17545306,
    "need_refresh": true,
    "version": 1775987047867
  }
}
```
</details>

#### 人气排行标签页变更

当主播切换直播分区时，由于所属分类发生变化，直播间前端 UI 上“人气排行榜”面板的分类标签（Tab）也需要随之重置。系统会下发此指令通知客户端刷新标签页。
（注：通常与 `CHG_RANK_REFRESH` 伴随出现，一个负责刷新底层数据，一个负责刷新前端 UI 标签）

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "POPULARITY_RANK_TAB_CHG" | 指示客户端刷新人气排行面板的标签页 UI |
| data | obj  | 标签页刷新参数 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| room_id | num | 直播间ID | 当前直播间的数字 ID |
| ruid | num | 主播UID | 当前直播间主播的 UID |
| type | str | 变更类型 | 触发标签页变更的排行类型。例如 `"area"` 代表**分区排行**的标签页需要刷新 |
| need_refresh_tab | bool | 是否刷新标签 | `true` 代表要求前端立刻执行排行标签页（Tab）的重绘动作 |

<details>
<summary>查看消息示例 (因切换分区触发)：</summary>

```json
{
  "cmd": "POPULARITY_RANK_TAB_CHG",
  "data": {
    "room_id": 3128551,
    "ruid": 17545306,
    "type": "area",
    "need_refresh_tab": true
  }
}
```
</details>

#### 直播间在人气榜的排名改变


json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "POPULAR_RANK_CHANGED" | |
| data | obj | 直播间的人气榜排名信息 | |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 主播UID | |
| rank | num | 人气榜排名 | |
| countdown | num | 人气榜下轮结算剩余时长 | |
| timestamp | num | 触发时的Unix时间戳 | |
| timestamp | str | 待调查 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    'cmd': 'POPULAR_RANK_CHANGED',
    'data': {
        'uid': 780791,
        'rank': 36,
        'countdown': 1927,
        'timestamp': 1702578474,
        'cache_key': 'rank_change:91a4e81ba3034ae894d61e432aa13081'
            }
}
```

</details>

#### PK礼物积分同步

在直播间进行连麦或 PK 状态时有人送礼，服务器会下发此包。V1 版本不仅包含实时的 PK 积分同步，还包含了极其详尽的前端 UI 渲染指令（如 WebRTC 的推流分辨率、播放器切割网格坐标等）。

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "UNIVERSAL_EVENT_GIFT" | 连麦状态下的礼物积分及 UI 布局全局同步事件。 |
| data | obj | 连麦状态完整详情 | 包含成员信息、积分信息与重度 UI 渲染数据。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| room_id | num | 当前直播间ID | |
| anchor_uid | num | 当前主播UID | |
| info | obj | 连麦会话的核心数据大包 | 包含所有连麦细节。 |

data.info字段 (连麦会话详情)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| biz_session_id | str | 连麦会话ID | 唯一标识当前这局连麦/PK的ID。 |
| interact_channel_id | str | 互动频道ID | |
| interact_mode | obj | 互动模式配置 | 包含邀请、连麦超时等参数。 |
| interact_template | obj | UI 互动模板引擎 | **核心渲染配置，包含客户端播放器分割布局。** |
| interact_connect_type | num | 互动连接类型 | 待调查，通常为 `0`。 |
| interact_max_users | num | 最大互动人数 | 例如 `9` 代表最多支持 9 人同屏连麦。 |
| members | array | 连麦成员列表 | **包含参与连麦的所有主播的信息（含对手）。** |
| version | num | 版本戳 | 通常为精确到毫秒的时间戳。 |
| session_status | num | 会话状态 | `1` 代表正在进行中。 |
| multi_conn_info | obj | 多人连接详情 | **核心数据：包含实时的 PK 分数。** |
| business_label | str | 业务标签 | 通常为 `"universal_multi_conn"` (通用多人连麦)。 |
| invoking_time | num | 调用次数/时间 | |
| members_version | num | 成员版本号 | |
| room_status | num | 房间状态 | `1` 代表正常。 |
| system_time_unix | num | 系统当前时间戳 | Unix时间戳（秒）。 |
| room_owner | num | 房主UID | 发起本次连麦/PK的房主UID。 |
| session_start_at | str | 会话开始时间 | 字符串格式的时间。 |
| session_start_at_ts | num | 会话开始时间戳 | |
| room_start_at | str | 房间开始时间 | 字符串格式的时间。 |
| room_start_at_ts | num | 房间开始时间戳 | |
| trace_id | str | 追踪ID | 用于后端链路追踪。 |

data.info.interact_mode字段 (互动规则配置)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| interact_mode_type | num | 模式类型 | |
| join_types | array | 允许加入的类型 | 如 `[1, 2]`。 |
| invite_timeout | num | 邀请超时时间 | 单位秒（如 `30`）。 |
| apply_timeout | num | 申请超时时间 | 单位秒（如 `20`）。 |
| position_mode | num | 占位模式 | |

data.info.interact_template字段 (UI 渲染引擎配置)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| template_id | str | 模板ID | 如 `"multi_conn_grid"`。 |
| is_variable_layout | bool | 是否可变布局 | |
| layout_list | array/null | 布局列表 | |
| show_interact_ui | bool | 是否显示互动UI | |
| layout_id | str | 当前布局ID | 如 `"left1_right1"` (左右分屏)。 |
| layout_data | obj | 布局详细坐标与参数 | 控制 Web/App 端的播放器切割。 |

data.info.interact_template.layout_data字段 (播放器切割参数)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| width | num | 网格总宽 | 例如 `10`。 |
| height | num | 网格总高 | 例如 `8`。 |
| default_cell | obj | 默认网格单元属性 | 包含默认的坐标、层级和各端字体大小配置。 |
| cells | array | 各画面的具体网格参数 | 根据成员人数分配画面的长宽坐标 `x`, `y`。 |
| rtc_resolution | obj | WebRTC 视频流分辨率 | 包含 `horizontal_width`, `horizontal_height`, 以及初始/最大/最小码率 (`code_rate`) 等音视频底层推流参数。 |
| best_area_show_pos | num | 最佳区域展示位 | 通常为 `-1`。 |

data.info.members数组 (连麦成员详情)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 主播UID | |
| uname | str | 主播昵称 | |
| face | str | 主播头像 | |
| position | num | 站位 | `0` 为左侧（通常为本直播间主播），`1` 为右侧（通常为对手）。 |
| join_time | num | 加入时间戳 | Unix时间戳（秒）。 |
| link_id | str | 连麦链接ID | |
| gender | num | 性别 | `0`女，`1`男，`-1`保密。 |
| room_id | num | 主播直播间长号 | |

data.info.multi_conn_info字段 (多人连接与积分)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| scores | array | **积分列表** | **存放各成员的实时 PK 分数**。 |
| room_owner | num | 房主UID | |
| show_score | num | 是否展示分数 | `1` 为展示。 |

data.info.multi_conn_info.scores数组 (实时积分表)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 对应主播的UID | |
| price | num | 后端真实积分 | 实际的礼物价值积分（通常为展示分数的 100 倍）。 |
| price_text | str | 前端展示积分 | 实际在 PK 条上显示的字符串（如 `"36"`）。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "UNIVERSAL_EVENT_GIFT",
  "data": {
    "room_id": 1950852193,
    "anchor_uid": 3546607121336514,
    "info": {
      "biz_session_id": "17748003749063546890274605289",
      "interact_channel_id": "4721463177254912",
      "interact_mode": {
        "interact_mode_type": 0,
        "join_types": [1, 2],
        "invite_timeout": 30,
        "apply_timeout": 20,
        "position_mode": 0
      },
      "interact_template": {
        "template_id": "multi_conn_grid",
        "is_variable_layout": true,
        "layout_list": null,
        "show_interact_ui": false,
        "layout_id": "left1_right1",
        "layout_data": {
          "width": 10,
          "height": 8,
          "default_cell": {
            "x": 0, "y": 0, "width": 5, "height": 8,
            "z_index": 0, "position": 0, "default_open": 1,
            "mobile_font_size": 12, "mobile_avatar_size": 64,
            "pc_web_font_size": 14, "pc_web_avatar_size": 112,
            "can_zoom": 0
          },
          "cells": [
            { "x": 0, "y": 0, "width": 0, "height": 0, "position": 0 },
            { "x": 5, "y": 0, "width": 0, "height": 0, "position": 1 }
          ],
          "rtc_resolution": {
            "vertical_width": 720, "vertical_height": 1152,
            "horizontal_width": 1000, "horizontal_height": 800,
            "code_rate_init": 1500, "code_rate_min": 800, "code_rate_max": 2000
          },
          "best_area_show_pos": -1
        }
      },
      "interact_connect_type": 0,
      "interact_max_users": 9,
      "members": [
        {
          "uid": 3546607121336514,
          "uname": "枳念念",
          "face": "[https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg](https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg)",
          "position": 0,
          "join_time": 1774800379,
          "link_id": "68942631",
          "gender": 0,
          "room_id": 1950852193
        },
        {
          "uid": 3546890274605289,
          "uname": "阿狸不吃梨lili",
          "face": "[https://i0.hdslb.com/bfs/face/b87c61ef3755bf2661ade8f9865bc9e09f572d18.jpg](https://i0.hdslb.com/bfs/face/b87c61ef3755bf2661ade8f9865bc9e09f572d18.jpg)",
          "position": 1,
          "join_time": 1774800379,
          "link_id": "68942630",
          "gender": -1,
          "room_id": 1940980374
        }
      ],
      "version": 1774800505417,
      "session_status": 1,
      "multi_conn_info": {
        "scores": [
          { "uid": 3546607121336514, "price": 100, "price_text": "1" },
          { "uid": 3546890274605289, "price": 0, "price_text": "0" }
        ],
        "room_owner": 3546890274605289,
        "show_score": 1
      },
      "business_label": "universal_multi_conn",
      "invoking_time": 1,
      "members_version": 3149586401,
      "room_status": 1,
      "system_time_unix": 1774800505,
      "room_owner": 3546890274605289,
      "session_start_at": "",
      "session_start_at_ts": 0,
      "room_start_at": "",
      "room_start_at_ts": 0,
      "trace_id": ""
    }
  }
}
```
</details>

#### PK礼物积分同步V2

当主播处于连麦或 PK 状态时，观众送礼会触发积分变动，服务器会下发此包以同步最新的分数和成员状态。V2 版本去除了冗余的 UI 布局数据，更加轻量化。但是现版本和 V1 是同时使用的

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "UNIVERSAL_EVENT_GIFT_V2" | 连麦状态下的礼物积分全局同步事件。 |
| data | obj | 连麦状态详情 | 包含成员信息、积分信息与基础 UI 标识。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| biz_session_id | str | 连麦会话ID | 唯一标识当前这局连麦/PK的ID。 |
| interact_channel_id | str | 互动频道ID | |
| interact_template | obj | UI 互动模板 | 包含基础的排版标识。 |
| members | array | 连麦成员列表 | **包含参与连麦的双方主播信息及实时积分。** |
| stream_control | obj/null | 流控信息 | 待调查 |
| version | num | 版本戳 | 通常为时间戳，用于前端按序处理包。 |
| session_status | num | 会话状态 | `1` 代表正在进行中。 |
| business_label | str | 业务标签 | 通常为 `"universal_multi_conn"` (通用多人连麦)。 |
| invoking_time | num | 调用次数/时间 | 待调查 |
| members_version | num | 成员版本号 | |
| room_status | num | 房间状态 | `1` 代表正常。 |
| system_time_unix | num | 系统当前时间戳 | Unix时间戳（秒）。 |
| room_owner | num | 房主UID | 发起本次连麦/PK的房主。 |
| session_start_at | str | 会话开始时间 | 格式如 `"2026-03-30 00:06:19"`。 |
| session_start_at_ts | num | 会话开始时间戳 | 相对或绝对时间戳标记。 |
| room_start_at | str | 房间开始时间 | 格式如 `"2026-03-30 00:06:19"`。 |
| room_start_at_ts | num | 房间开始时间戳 | |
| trace_id | str | 追踪ID | 用于后端链路追踪的 UUID。 |
| biz_extra_data | obj | 业务附加数据(全局) | 包含全局显示配置。 |
| channel_users | array | 频道用户UID列表 | 参与连麦的所有主播 UID 数组。 |

data.interact_template字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| template_id | str | 模板ID | 如 `"multi_conn_grid"`。 |
| show_interact_ui | bool | 是否显示互动UI | |
| layout_id | str | 布局ID | 如 `"left1_right1"` 代表左右分屏。 |
| layout_version | num | 布局版本号 | |

data.members数组 (连麦成员详情)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 主播UID | |
| uname | str | 主播真实昵称 | |
| face | str | 主播头像URL | |
| position | num | 站位 | `0` 为左侧（通常为本直播间主播），`1` 为右侧（对手）。 |
| join_time | num | 加入时间戳 | |
| link_id | str | 连麦链接ID | |
| gender | num | 性别 | `0`女，`1`男，`-1`保密。 |
| room_id | num | 主播直播间长号 | |
| fans_num | num | 粉丝数 | 在此包中通常为 `0`，待核实。 |
| display_name | str | 展示名称 | 本房主播通常显示为 `"本房主播"`，对手显示真实昵称。 |
| biz_extra_data | obj | 业务附加数据(个人) | **内含个人的实时 PK 积分**。 |
| join_time_ts | num | 加入时间戳(备用) | |

data.members.biz_extra_data.multi_conn字段 (个人实时积分)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| price | num | 后端真实积分 | 实际的礼物价值积分（通常为展示分数的 100 倍）。 |
| price_text | str | 前端展示积分 | 实际在 PK 条上显示的字符串（如 `"1"`）。 |

data.biz_extra_data.multi_conn字段 (全局附加配置)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| show_score | num | 是否展示分数 | `1` 为展示。 |
| support_full_zoom | num | 支持全屏放大 | 待调查。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "UNIVERSAL_EVENT_GIFT_V2",
  "data": {
    "biz_session_id": "17748003749063546890274605289",
    "interact_channel_id": "4721463177254912",
    "interact_template": {
      "template_id": "multi_conn_grid",
      "show_interact_ui": false,
      "layout_id": "left1_right1",
      "layout_version": 14
    },
    "members": [
      {
        "uid": 3546607121336514,
        "uname": "枳念念",
        "face": "[https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg](https://i0.hdslb.com/bfs/face/f45351c32455b2c7246a07d206acc51f4a8671ff.jpg)",
        "position": 0,
        "join_time": 1774800379,
        "link_id": "68942631",
        "gender": 0,
        "room_id": 1950852193,
        "fans_num": 0,
        "display_name": "本房主播",
        "biz_extra_data": {
          "multi_conn": {
            "price": 100,
            "price_text": "1"
          }
        },
        "join_time_ts": 0
      }
    ],
    "stream_control": null,
    "version": 1774800505415,
    "session_status": 1,
    "business_label": "universal_multi_conn",
    "invoking_time": 2,
    "members_version": 3358334162,
    "room_status": 1,
    "system_time_unix": 1774800505,
    "room_owner": 3546890274605289,
    "session_start_at": "2026-03-30 00:06:19",
    "session_start_at_ts": 126,
    "room_start_at": "2026-03-30 00:06:19",
    "room_start_at_ts": 126,
    "trace_id": "531f5a86d6aec3ba09e7b1f91669c94e",
    "biz_extra_data": {
      "multi_conn": {
        "show_score": 1,
        "support_full_zoom": 2
      }
    },
    "channel_users": [
      3546607121336514,
      3546890274605289
    ]
  }
}
```
</details>

#### PK战况与MVP实时更新

在 PK 过程中，用于高频同步双方的总票数以及贡献榜单（MVP 大哥榜）。通常包含发起方 (`init_info`) 和匹配方 (`match_info`) 两个阵营的实时战况。

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "PK_BATTLE_PROCESS_NEW" | PK 进程实时战报。 |
| data | obj | 战况详细数据 | 包含双方阵营的分数和助攻榜。 |
| msg_id | str | 消息序列号 | |
| p_is_ack | bool | 是否需要回执 | |
| p_msg_type | num | 消息协议类型 | |
| pk_id | num | PK 唯一标识符 | 对应本局 PK 的全局 ID。 |
| pk_status | num | PK 状态 | `201` 通常代表进行中。 |
| send_time | num | 发送时间戳 | 毫秒级 Unix 时间戳。 |
| template_id | str | 模板ID | 如 `"multi_conn_grid"`。 |
| timestamp | num | 时间戳 | 秒级 Unix 时间戳。 |
| trace_id | str | 追踪ID | 用于后端链路追踪的 UUID。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| battle_type | num | 战斗类型 | 待调查（例如 `6`）。 |
| init_info | obj | 发起方战况 | PK 发起方主播的实时阵营数据。 |
| match_info | obj | 匹配方战况 | PK 接受方（匹配方）主播的实时阵营数据。 |
| trace_id | str | 追踪ID | 内部链路追踪。 |

data.init_info / data.match_info 字段 (双方阵营战况结构相同)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| room_id | num | 该阵营的直播间长号 | 用于区分这是哪边主播的数据。 |
| votes | num | 当前总票数 | 该阵营的实时 PK 总分。 |
| best_uname | str | 贡献最高的用户名 | 当前的 MVP 大哥昵称。 |
| assist_info | array/null | 助攻榜单 (MVP列表) | 包含贡献最高用户的详细数组，暂无贡献时为 `null`。 |
| vision_desc | num | 视觉描述标识 | 待调查（例如 `0`）。 |

assist_info 数组 (助攻大哥详细信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| rank | num | 排名 | 该用户在本次 PK 助攻榜的当前排名（如 `1` 代表榜一）。 |
| uid | num | 用户UID | |
| uname | str | 用户昵称 | |
| face | str | 头像URL | |
| is_mystery | bool | 是否神秘人 | |
| award_content | str | 奖励内容 | 待调查。 |
| uinfo | obj | **大哥的详细底层信息** | **结构与 `ENTRY_EFFECT` 中的 `uinfo` 完全一致，包含精确的粉丝牌、财富等级等数据。** |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "PK_BATTLE_PROCESS_NEW",
  "data": {
    "battle_type": 6,
    "init_info": {
      "assist_info": null,
      "best_uname": "",
      "room_id": 1940980374,
      "vision_desc": 0,
      "votes": 0
    },
    "match_info": {
      "assist_info": [
        {
          "award_content": "",
          "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
          "is_mystery": false,
          "rank": 1,
          "uid": 26928797,
          "uinfo": {
            "base": {
              "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
              "is_mystery": false,
              "name": "mikufilck",
              "name_color": 0,
              "name_color_str": "",
              "official_info": {
                "desc": "",
                "role": 0,
                "title": "",
                "type": -1
              },
              "origin_info": {
                "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
                "name": "mikufilck"
              },
              "risk_ctrl_info": null
            },
            "guard": null,
            "guard_leader": null,
            "medal": null,
            "title": null,
            "uhead_frame": null,
            "uid": 26928797,
            "wealth": null
          },
          "uname": "mikufilck"
        }
      ],
      "best_uname": "mikufilck",
      "room_id": 1950852193,
      "vision_desc": 0,
      "votes": 1
    },
    "trace_id": "531f5a86d6aec3ba09e7b1f91669c94e"
  },
  "msg_id": "90436141516382208:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "pk_id": 393724504,
  "pk_status": 201,
  "send_time": 1774800505439,
  "template_id": "multi_conn_grid",
  "timestamp": 1774800505,
  "trace_id": "531f5a86d6aec3ba09e7b1f91669c94e"
}
```
</details>

#### PK全局状态

包含当前 PK 对局的所有元数据，权限极高。通常用于让前端校准倒计时时钟、确认惩罚阶段的时间点、拉取双方连胜记录、加载特殊的 UI 颜色配置，以及处理逃跑/异常中断状态。

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "PK_INFO" | PK 全局元数据大包。 |
| data | obj | PK 详细信息 | 包含时间轴、双方阵营大全、玩法配置等。 |
| msg_id | str | 消息序列号 | |
| p_is_ack | bool | 是否需要回执 | |
| p_msg_type | num | 消息协议类型 | |
| send_time | num | 发送时间戳 | 毫秒级 Unix 时间戳。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| audience_open | bool | 观众面板是否开启 | |
| invite_pk_resp | obj/null | 邀请回复信息 | |
| members | array | 双方详细阵营数据 | **核心节点，包含双方的完整票数、连胜记录和助攻榜。** |
| pk_basic | obj | PK 核心时间轴 | **核心节点，控制比赛的生死倒计时。** |
| pk_group | obj/null | PK 群组信息 | |
| pk_match_info | obj/null | PK 匹配信息 | |
| pk_play | obj | 玩法与UI配置 | 包含前端渲染特效、弹幕颜色及逃跑控制配置。 |
| mill_timestamp | num | 毫秒级时间戳 | |
| timestamp | num | 秒级时间戳 | |

data.members数组 (阵营详细数据)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 阵营主播UID | |
| uname | str | 阵营主播昵称 | |
| face | str | 阵营主播头像 | |
| room_id | num | 阵营直播间号 | |
| rank | num | 当前排名 | 如 `1` 为领先方，`2` 为落后方。 |
| rank_v2 | num | 新版排名 | |
| votes | num | 阵营总票数 | 真实 PK 分数。 |
| votes_text | str | 展示票数 | |
| golds | num | 获得金币/积分数 | |
| assist_info | array | 助攻榜单 (MVP) | 包含贡献最高的大哥数组，格式与 `PK_BATTLE_PROCESS_NEW` 一致。 |
| battle_level | obj | 战斗等级/挂件 | 包含 `icon` 和挂件交互的 H5 `url`。 |
| date_streak | num | 连胜次数 | |
| is_latest_streak | bool | 是否正在连胜 | |
| is_winner | num | 获胜标志 | `0` 暂无结果，`1` 获胜。 |
| status | num | 阵营状态 | |
| capsules_v2 | obj/null | 新版胶囊特效 | |
| group_id | num | 分组ID | |
| is_follow | num | 是否已关注对手 | |
| order | num | 排序权重 | |
| pk_cards | obj/null | PK专属道具卡 | 如暴击卡、护盾等数据。 |
| pk_multiple_status | num | 多倍积分状态 | |
| play | obj/null | 玩法数据 | |
| power | str | 战力值 | 可能用于特殊的战力比拼玩法。 |

data.pk_basic字段 (赛事核心时间轴与属性)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| pk_id | num | 本局 PK ID | 全局唯一标识符。 |
| biz_session_id | str | 连麦会话ID | 匹配底层的连麦链路。 |
| init_id | num | 发起方房间号 | |
| init_uid | num | 发起方UID | |
| start_time | num | PK 开始时间 | Unix 时间戳 (秒)。 |
| end_time | num | PK 结束时间 | Unix 时间戳 (秒)，**前端用此计算比赛剩余倒计时。** |
| punish_end_time | num | 惩罚结束时间 | Unix 时间戳 (秒)，**失败方接受惩罚的截止时间。** |
| status | num | PK 状态码 | `201` 为进行中，`401` 为正常结算与惩罚倒计时，`601` 为PK合流失败，`610`为逃跑/异常断开， `1001` 为彻底关闭。 |
| status_msg | str | 状态提示文案 | 异常时的文字提示，如 `"PK合流失败，请重新进行匹配"`。 |
| punish_text | str | 惩罚文本 | 通常为 `"惩罚"`。 |
| main_page | str | H5活动页URL | |
| season_id | num | 当前赛季ID | |
| sprint_duration | num | 冲刺阶段时长 | |
| template_id | str | UI模板ID | 如 `"multi_conn_grid"`。 |
| type | num | PK大类 | |
| sub_type | num | PK子玩法类型 | |
| muti_pk_type | num | 多人PK类型 | 例如 `3` 或 `4`。 |
| satellite_info | obj/null | 待调查 | |

data.members.capsules_v2数组 (PK 限时挑战特效胶囊)

当 PK 过程中触发了限时翻倍、连击任务等局内挑战时，此数组会包含相关的倒计时和提示信息，用于前端渲染悬浮的胶囊横幅。

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| id | str | 胶囊ID | 如 `"1"`，`"2"`，或 `"10"` |
| text | str | 提示文本 | 如 `"完成挑战获得<%2倍%>PK值"`、`"差1人翻倍"` 或 `"挑战失败"` |
| capsule_type | num | 胶囊类型 | 例如 `4` 或 `5` |
| biz_style_type | num | 业务样式类型 | 决定前端 UI 的渲染样式（如颜色或闪烁特效） |
| end_time | num | 截止时间戳 | 秒级。限时任务结束的时间，为 `0` 代表无倒计时 |
| progress | num | 进度 | 任务当前进度 |
| url | str | 跳转链接 | 包含交互协议（如 `bilibili://live/...`）用于点击唤起面板 |
| animation_text | str | 动画文本 | 待调查 |
| blink_event_type | num | 闪烁事件类型 | 待调查 |
| show_terminal | num | 显示端标识 | 待调查 |
| toast | str | 浮窗提示 | 待调查 |

data.pk_play字段 (UI与玩法特效配置)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| dm_conf | obj | 弹幕配置 | 包含 `bg_color` (背景色) 和 `font_color` (字体色)。 |
| pre_duration | num | 准备时长 | 如 `10` 秒。 |
| show_streak | bool | 是否显示连胜特效 | |
| escape | obj | 逃跑控制与提示 | 包含 `count` (逃跑次数), `puni_time` (惩罚时间), 以及 `tips` (提示语，如 `"是否要提前结束PK?"`)。 |
| final_conf | obj | 决战时刻配置 | 最后冲刺阶段的配置 (`start_time`, `end_time`, `switch`)。 |
| pk_score_multiple_play | obj/null | PK积分翻倍配置 | 翻倍玩法相关。 |

<details>
<summary>查看消息示例 (包含逃跑提示的情况)：</summary>

```json
{
  "cmd": "PK_INFO",
  "data": {
    "audience_open": false,
    "invite_pk_resp": null,
    "members": [
      {
        "assist_info": [
          {
            "award_content": "",
            "face": "[https://i0.hdslb.com/bfs/face/be1dd12a0c5dbd296bdf09246b6e5ee5093c0324.jpg](https://i0.hdslb.com/bfs/face/be1dd12a0c5dbd296bdf09246b6e5ee5093c0324.jpg)",
            "is_mystery": false,
            "rank": 1,
            "uid": 36711753,
            "uname": "橘町w"
          }
        ],
        "battle_level": {
          "icon": "[https://i0.hdslb.com/bfs/live/4022aa441d1bcdd2f98ac1f15767f5c8994be01d.png](https://i0.hdslb.com/bfs/live/4022aa441d1bcdd2f98ac1f15767f5c8994be01d.png)",
          "url": "[https://live.bilibili.com/activity/live-activity-battle/index.html?room_id=24538659](https://live.bilibili.com/activity/live-activity-battle/index.html?room_id=24538659)..."
        },
        "capsules": null,
        "capsules_v2": null,
        "date_streak": 0,
        "face": "[https://i1.hdslb.com/bfs/face/a415ce881066fb8f71132253effcdde62540bc05.jpg](https://i1.hdslb.com/bfs/face/a415ce881066fb8f71132253effcdde62540bc05.jpg)",
        "golds": 108100,
        "group_id": 0,
        "is_follow": 0,
        "is_latest_streak": false,
        "is_winner": 1,
        "order": 0,
        "pk_cards": null,
        "pk_multiple_status": 0,
        "play": null,
        "power": "",
        "rank": 1,
        "rank_v2": 0,
        "room_id": 24538659,
        "status": 3,
        "uid": 512033026,
        "uname": "西妮贝尔",
        "votes": 1981,
        "votes_text": "1981"
      }
    ],
    "mill_timestamp": 1776343803553,
    "pk_basic": {
      "biz_session_id": "1776343426074512033026",
      "end_time": 1776343736,
      "init_id": 24538659,
      "init_uid": 512033026,
      "main_page": "[https://live.bilibili.com/activity/live-activity-battle/index.html](https://live.bilibili.com/activity/live-activity-battle/index.html)...",
      "muti_pk_type": 3,
      "pk_id": 394393841,
      "punish_end_time": 1776343799,
      "punish_text": "惩罚",
      "satellite_info": null,
      "season_id": 96,
      "sprint_duration": 10,
      "start_time": 1776343426,
      "status": 1001,
      "status_msg": "",
      "sub_type": 8,
      "template_id": "multi_conn_grid",
      "type": 2
    },
    "pk_group": null,
    "pk_match_info": null,
    "pk_play": {
      "dm_conf": {
        "bg_color": "#72C5E2",
        "font_color": "#FFE10B"
      },
      "escape": {
        "count": 0,
        "puni_time": 0,
        "tips": "是否要提前结束PK?"
      },
      "final_conf": {
        "end_time": 0,
        "start_time": 0,
        "switch": 0
      },
      "pk_score_multiple_play": null,
      "pre_duration": 10,
      "show_streak": false
    },
    "timestamp": 1776343803
  },
  "msg_id": "92054406934093824:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "send_time": 1776343803605
}
```
</details>

#### PK惩罚战斗结束

当 PK 正常结束并度过惩罚倒计时，或者一方强制中断/逃跑导致 PK 提前结束时，服务器会广播此指令清理状态。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "PK_BATTLE_PUNISH_END" | 指示 PK 阶段彻底结束，准备恢复常规直播间 UI |
| data | obj  | 战斗类型数据 | 见下方展开 |
| msg_id | str | 消息序列号 | |
| p_is_ack | bool | 是否需要回执 | |
| p_msg_type | num | 消息协议类型 | |
| pk_id | num | PK 唯一标识 | |
| pk_status | num | 结束状态码 | `1001` 通常代表非正常/提前终止 |
| status_msg | str | 状态信息 | |
| template_id | str | UI 模板ID | 如 `"multi_conn_grid"` |
| send_time | num | 发送时间戳 | 毫秒级 |
| timestamp | num | 时间戳 | 秒级 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| battle_type | num | 战斗大类 | |
| battle_sub_type | num | 战斗子类 | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "PK_BATTLE_PUNISH_END",
  "data": {
    "battle_sub_type": 0,
    "battle_type": 2
  },
  "msg_id": "92054406915172352:1000:1000",
  "p_is_ack": true,
  "p_msg_type": 1,
  "pk_id": 394393841,
  "pk_status": 1001,
  "send_time": 1776343803587,
  "status_msg": "",
  "template_id": "multi_conn_grid",
  "timestamp": 1776343803
}
```
</details>

#### PK战斗结算与惩罚

当 PK 倒计时归零，进入结算和惩罚阶段时下发此指令。包含双方的最终比分、胜负判定（`result_type`）以及获胜方的大哥助攻榜（MVP）。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "PK_BATTLE_SETTLE_NEW" | 指示 PK 进入结算阶段 |
| data | obj  | 结算大包 | 包含最终胜负判定与惩罚倒计时。 |
| msg_id | str | 消息序列号 | |
| pk_id | num | PK 唯一标识 | |
| pk_status | num | PK 状态码 | 如 `401` 代表结算/惩罚阶段 |
| template_id | str | UI 模板ID | 如 `"multi_conn_grid"` |
| send_time | num | 发送时间戳 | 毫秒级 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| battle_type | num | 战斗类型 | 例如 `6` |
| dm_conf | obj | 弹幕配置 | 包含 `bg_color` 和 `font_color` |
| dmscore | num | 弹幕权重 | |
| init_info | obj | 发起方结算信息 | 包含最终票数和胜负判定。 |
| match_info | obj | 匹配方结算信息 | 包含最终票数、胜负判定以及助攻榜单（MVP）。 |
| pk_id | num | PK 唯一标识 | |
| pk_status | num | PK 状态码 | |
| punish_end_time | num | 惩罚结束时间戳 | 秒级时间戳，前端用此展示惩罚倒计时 |
| punish_name | str | 惩罚阶段名称 | 通常为 `"惩罚"` |
| settle_status | num | 结算状态 | `1` 代表已结算 |
| timestamp | num | 结算时间戳 | 秒级 |

data.init_info / data.match_info (双方结算详情)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| result_type | num | 胜负判定 | 核心字段！用于判定该阵营的胜负（例如 `2` 代表获胜，`-1` 或 `0` 代表失败/平局，待详细统计核实）。 |
| room_id | num | 直播间长号 | |
| votes | num | 最终票数 | |
| assist_info | array/null | 助攻榜单(MVP) | 包含详细的 `uinfo` 数据，结构与 `PK_BATTLE_PROCESS_NEW` 一致。失败方通常为 `null`。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "PK_BATTLE_SETTLE_NEW",
  "data": {
    "battle_type": 6,
    "dm_conf": {
      "bg_color": "#72C5E2",
      "font_color": "#FFE10B"
    },
    "dmscore": 1008,
    "init_info": {
      "assist_info": null,
      "result_type": -1,
      "room_id": 23856200,
      "votes": 0
    },
    "match_info": {
      "assist_info": [
        {
          "rank": 1,
          "uid": 26928797,
          "uname": "mikufilck",
          "uinfo": { "...": "..." }
        }
      ],
      "result_type": 2,
      "room_id": 24538659,
      "votes": 1
    },
    "pk_id": 394394669,
    "pk_status": 401,
    "punish_end_time": 1776344654,
    "punish_name": "惩罚",
    "settle_status": 1,
    "timestamp": 1776344596
  }
}
```
</details>

#### PK任务进度挂件

用于在直播间画面中展示类似“大乱斗/PK 日常任务”进度的悬浮挂件 UI（例如：完成 X 场 PK 即可获得段位分奖励）。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "PK_WIDGET" | 指示更新 PK 任务进度挂件 |
| data | obj  | 挂件渲染参数 | 包含任务详情和显示开关。 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| show | bool | 是否显示挂件 | `true` 显示，`false` 隐藏 |
| title | str | 挂件标题 | |
| text | str | 挂件文本 | |
| task | obj | 任务详情 | 见下方展开 |

data.task字段 (任务详情)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 任务名称 | 如 `"PK场次"` |
| current_num | num | 当前进度 | 当前已完成的场次/进度（如 `1`） |
| need_num | num | 目标进度 | 需要达到的总场次/进度（如 `6`） |
| reward | str | 任务奖励文本 | 如 `"大乱斗段位分+10分"` |
| icon | str | 任务图标 URL | |
| task_type | num | 任务类型 | 待调查（例如 `2`） |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "PK_WIDGET",
  "data": {
    "task": {
      "name": "PK场次",
      "need_num": 6,
      "current_num": 1,
      "icon": "[https://i0.hdslb.com/bfs/live/2c4e31c69ced39a4f5e6e93d36ce5af28bf0d77a.png](https://i0.hdslb.com/bfs/live/2c4e31c69ced39a4f5e6e93d36ce5af28bf0d77a.png)",
      "reward": "大乱斗段位分+10分",
      "task_type": 2
    },
    "title": "",
    "text": "",
    "show": true
  }
}
```
</details>

#### 直播间系统吐司提示

服务器向当前直播间下发的全局系统级文字提示。客户端收到后通常会在画面中央或底部以半透明黑底白字的形式（Toast）弹出提示框。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "LIVE_ROOM_TOAST_MESSAGE" | 系统级的 Toast 文本提示指令 |
| data | obj  | 提示详情 | 见下方展开 |
| timestamp | num | 时间戳 | 外层秒级时间戳 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| message | str | 提示文本 | **核心内容**。如 `"对方主播结束了视频连线"`，前端直接展示此文本。 |
| timestamp | num | 时间戳 | 内层秒级时间戳 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "LIVE_ROOM_TOAST_MESSAGE",
  "timestamp": 1776343809,
  "data": {
    "timestamp": 1776343809,
    "message": "对方主播结束了视频连线"
  }
}
```
</details>

#### 直播间用户点赞

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "LIKE_INFO_V3_CLICK" | 若直播间被赞，则内容是"LIKE_INFO_V3_CLICK" |
| data | obj | 点赞用户信息与点赞细节 | 包含了统一的 `uinfo` 底层结构。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 点赞用户的UID | |
| uname | str | 点赞用户的名称 | 建议优先使用 `uinfo.base.name`。 |
| uname_color | str | 点赞用户的名称颜色 | |
| like_icon | str | 点赞飘屏图标的URL | 前端右下角飘起的图标样式。 |
| like_text | str | 点赞动作文本 | 通常为 `"为主播点赞了"`。 |
| msg_type | num | 消息类型 | 默认为 `6`。 |
| show_area | num | 显示区域 | 待调查（例如 `1`）。 |
| is_mystery | bool | 是否神秘人 | |
| dmscore | num | 互动积分 | |
| identities | array | 身份组 | 例如 `[3, 1]`，代表用户的某些特权标识。 |
| contribution_info | obj | 贡献信息 | 包含 `grade` (等级) 等字段。 |
| group_medal | obj/null | 粉丝团信息 | 待调查，通常为 null。 |
| fans_medal | obj | **旧版粉丝勋章信息** | **点赞用户当前佩戴的粉丝牌（扁平结构）。** |
| uinfo | obj | **点赞用户的底层信息** | **包含精确的 `base`, `medal`, `guard` 等新版结构字段。** |

data.fans_medal字段 (旧版粉丝勋章)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| target_id | num | 目标UID | 发放该粉丝牌的主播 UID。 |
| medal_name | str | 勋章名称 | |
| medal_level | num | 勋章等级 | |
| medal_color | num | 基础颜色 | 十进制颜色值。 |
| medal_color_start | num | 渐变起始色 | 十进制颜色值。 |
| medal_color_end | num | 渐变结束色 | 十进制颜色值。 |
| medal_color_border | num | 边框颜色 | 十进制颜色值。 |
| is_lighted | num | 是否点亮 | `1`点亮，`0`熄灭。 |
| guard_level | num | 大航海等级 | `0`无，`1`总督，`2`提督，`3`舰长。 |
| icon_id | num | 图标ID | |
| anchor_roomid | num | 主播房间号 | 发放该粉丝牌的直播间号（部分情况为 0）。 |
| score | num | 亲密度/积分 | |
| special | str | 特殊标识 | |

data.uinfo.base字段 (新版基础信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| name | str | 用户完整昵称 | |
| face | str | 头像URL | |
| is_mystery | bool | 是否隐藏身份 | |
| name_color | num | 名字颜色 | |
| name_color_str | str | 名字颜色字符串 | |
| official_info | obj | 官方认证信息 | 包含 `desc`, `role`, `title`, `type` (普通用户 type 通常为 -1)。 |
| origin_info | obj | 原始信息 | 包含 `face` 和 `name` 的备份。 |
| risk_ctrl_info | obj/null | 风控相关信息 | |

data.uinfo.medal字段 (新版粉丝勋章)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| name | str | 粉丝勋章名称 | |
| level | num | 粉丝勋章等级 | |
| color | num | 基础颜色 | |
| color_start | num | 背景渐变起始色 | |
| color_end | num | 背景渐变结束色 | |
| color_border | num | 边框颜色 | |
| id | num | 粉丝牌ID | |
| typ | num | 待调查 | |
| is_light | num | 粉丝牌是否点亮 | `1`点亮，`0`熄灭。 |
| ruid | num | 发放此粉丝牌的主播UID | |
| guard_level | num | 对应的大航海等级 | `0`无，`1`总督，`2`提督，`3`舰长。 |
| score | num | 亲密度/积分 | |
| guard_icon | str | 舰队图标 | |
| honor_icon | str | 荣誉图标 | |
| user_receive_count | num | 待调查 | |
| v2_medal_color_start | str | V2起始色 | Hex格式（带透明度），如 `"#3FB4F699"`。 |
| v2_medal_color_end | str | V2结束色 | Hex格式（带透明度）。 |
| v2_medal_color_border | str | V2边框色 | Hex格式（带透明度）。 |
| v2_medal_color_text | str | V2文字色 | Hex格式，如 `"#FFFFFF"`。 |
| v2_medal_color_level | str | V2等级色 | Hex格式（带透明度）。 |

data.uinfo.guard字段 (大航海/舰队)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| level | num | 大航海等级 | `0`无，`1`总督，`2`提督，`3`舰长。 |
| expired_str | str | 过期时间 | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "LIKE_INFO_V3_CLICK",
  "data": {
    "contribution_info": {
      "grade": 0
    },
    "dmscore": 128,
    "fans_medal": {
      "anchor_roomid": 0,
      "guard_level": 0,
      "icon_id": 0,
      "is_lighted": 1,
      "medal_color": 1725515,
      "medal_color_border": 1725515,
      "medal_color_end": 5414290,
      "medal_color_start": 1725515,
      "medal_level": 24,
      "medal_name": "啵莉星",
      "score": 6003,
      "special": "",
      "target_id": 3546792096434666
    },
    "group_medal": null,
    "identities": [3, 1],
    "is_mystery": false,
    "like_icon": "[https://i0.hdslb.com/bfs/live/23678e3d90402bea6a65251b3e728044c21b1f0f.png](https://i0.hdslb.com/bfs/live/23678e3d90402bea6a65251b3e728044c21b1f0f.png)",
    "like_text": "为主播点赞了",
    "msg_type": 6,
    "show_area": 1,
    "uid": 26928797,
    "uinfo": {
      "base": {
        "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
        "is_mystery": false,
        "name": "mikufilck",
        "name_color": 0,
        "name_color_str": "",
        "official_info": {
          "desc": "",
          "role": 0,
          "title": "",
          "type": -1
        },
        "origin_info": {
          "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
          "name": "mikufilck"
        },
        "risk_ctrl_info": null
      },
      "guard": {
        "expired_str": "",
        "level": 0
      },
      "guard_leader": null,
      "medal": {
        "color": 1725515,
        "color_border": 1725515,
        "color_end": 5414290,
        "color_start": 1725515,
        "guard_icon": "",
        "guard_level": 0,
        "honor_icon": "",
        "id": 0,
        "is_light": 1,
        "level": 24,
        "name": "啵莉星",
        "ruid": 3546792096434666,
        "score": 6003,
        "typ": 0,
        "user_receive_count": 0,
        "v2_medal_color_border": "#3FB4F699",
        "v2_medal_color_end": "#3FB4F699",
        "v2_medal_color_level": "#3FB4F6E6",
        "v2_medal_color_start": "#3FB4F699",
        "v2_medal_color_text": "#FFFFFF"
      },
      "title": null,
      "uhead_frame": null,
      "uid": 26928797,
      "wealth": null
    },
    "uname": "mikufilck",
    "uname_color": ""
  }
}
```

</details>

#### 直播间点赞数

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "LIKE_INFO_V3_UPDATE" | 若直播间点赞数更新，则内容是"LIKE_INFO_V3_UPDATE" |
| data | obj | 直播间点赞数 | |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| click_count | num | 直播间点赞数 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "LIKE_INFO_V3_UPDATE",
    "data": {
        "click_count": 3227
    }
}
```

</details>

#### 直播间发红包弹幕

当有老板发送红包，直播间公屏弹出“老板大气！点点红包抽礼物”等参与弹幕，并出现红包抽奖UI时触发。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "POPULARITY_RED_POCKET_START"或"POPULARITY_RED_POCKET_V2_START" | 触发红包抽奖开始的弹幕和UI的数据包，V2与V1结构基本一致，通常同时发送以兼容新旧系统 |
| data | obj  | 送红包的老板的信息、红包内的礼物信息及红包倒计时状态 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| anchor_h5_url | str | 主播端H5页面链接 | **【注意】这是 V2 版本与 V1 版本唯一不同的地方。**在V1中此字段通常为空字符串，在V2中包含了用于控制主播端或新版移动端UI参数（如半屏Webview）的具体URL。 |
| animation_icon_url | str | 动画图标URL | 若无则为空字符串 |
| awards | array | 红包内包含的奖品（礼物）信息列表 | 见下方展开 |
| current_time | num | 服务器发送数据包的Unix时间戳 | |
| danmu | str | 参与弹幕内容 | 用户参与红包抽奖时自动发送的弹幕内容 |
| end_time | num | 抢红包结束的Unix时间戳 | |
| h5_url | str | 红包活动的H5页面链接 | |
| icon_url | str | 图标URL | 若无则为空字符串 |
| is_mystery | bool | 是否神秘人 | |
| join_requirement | num | 参与条件 | 待调查 |
| last_time | num | 红包持续时间 | 单位：秒 (通常为 start_time - end_time) |
| lot_config_id | num | 抽奖配置ID | 待调查 |
| lot_id | num | 发送的红包抽奖唯一ID | |
| lot_status | num | 红包状态 | 待调查 |
| remove_time | num | 移除倒计时UI的时间戳 | |
| replace_time | num | 替换时间戳 | 待调查 |
| rp_guard_info | obj | 舰队红包信息 | 若无则为null |
| rp_type | num | 红包类型 | |
| sender_face | str | 发送者的头像URL | |
| sender_name | str | 发送者的名称 | |
| sender_uid | num | 发送者的UID | |
| sender_uinfo | obj | 发送者详细资料 | 包含头像、认证等全套数据，见下方展开 |
| start_time | num | 开始抢红包的Unix时间戳 | |
| total_price | num | 红包总价格 | |
| user_status | num | 用户参与状态 | 1已参与，2未参与 |
| wait_num | num | 待调查 | |
| wait_num_v2 | num | 待调查 | |

data的awards数组中的对象

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_id | num | 礼物ID | |
| gift_name | str | 礼物名称 | |
| gift_pic | str | 礼物图标URL | |
| num | num | 该奖品的数量 | |

data的sender_uinfo字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| uid | num | 发送者UID | |
| base | obj | 基础资料 | 见下方展开 |
| medal | obj | 勋章详情 | 若无则为null |
| wealth | obj | 财富信息 | 若无则为null |
| title | obj | 称号信息 | 若无则为null |
| guard | obj | 舰队信息 | 若无则为null |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| guard_leader | obj | 舰队队长状态 | 若无则为null |

data的sender_uinfo的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 用户昵称 | |
| face | str | 头像链接 | |
| name_color | num | 昵称颜色 | |
| is_mystery | bool | 是否神秘人 | |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |
| origin_info | obj | 原始信息 | 见下方展开 |
| official_info | obj | 认证信息 | 见下方展开 |
| name_color_str | str | 昵称颜色字符串 | |

data的sender_uinfo的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 原始昵称 | |
| face | str | 原始头像链接 | |

data的sender_uinfo的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| role | num | 角色类型 | |
| title | str | 认证头衔 | |
| desc | str | 认证描述 | |
| type | num | 认证类型 | |


<details>
<summary>查看消息示例 (V2版)：</summary>

```json
{
  "cmd": "POPULARITY_RED_POCKET_V2_START",
  "data": {
    "lot_id": 27867696,
    "sender_uid": 26928797,
    "sender_name": "mikufilck",
    "sender_face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
    "join_requirement": 1,
    "danmu": "老板大气！点点红包抽礼物",
    "current_time": 1775820034,
    "start_time": 1775820033,
    "end_time": 1775820333,
    "last_time": 300,
    "remove_time": 1775820348,
    "replace_time": 1775820343,
    "lot_status": 1,
    "h5_url": "[https://live.bilibili.com/p/html/live-app-red-envelope/popularity.html?is_live_half_webview=1&hybrid_half_ui=1,5,100p,100p,000000,0,50,0,0,1;2,5,100p,100p,000000,0,50,0,0,1;3,5,100p,100p,000000,0,50,0,0,1;4,5,100p,100p,000000,0,50,0,0,1;5,5,100p,100p,000000,0,50,0,0,1;6,5,100p,100p,000000,0,50,0,0,1;7,5,100p,100p,000000,0,50,0,0,1;8,5,100p,100p,000000,0,50,0,0,1&hybrid_rotate_d=1&hybrid_biz=popularityRedPacket&lotteryId=27867696](https://live.bilibili.com/p/html/live-app-red-envelope/popularity.html?is_live_half_webview=1&hybrid_half_ui=1,5,100p,100p,000000,0,50,0,0,1;2,5,100p,100p,000000,0,50,0,0,1;3,5,100p,100p,000000,0,50,0,0,1;4,5,100p,100p,000000,0,50,0,0,1;5,5,100p,100p,000000,0,50,0,0,1;6,5,100p,100p,000000,0,50,0,0,1;7,5,100p,100p,000000,0,50,0,0,1;8,5,100p,100p,000000,0,50,0,0,1&hybrid_rotate_d=1&hybrid_biz=popularityRedPacket&lotteryId=27867696)",
    "anchor_h5_url": "[https://live.bilibili.com/p/html/live-app-red-envelope/popularitygiftlist.html?is_live_half_webview=1&hybrid_rotate_d=1&hybrid_biz=popularityRedPacket&is_cling_player=1&hybrid_half_ui=1,3,100p,70p,0,0,30,100,12;2,2,375,100p,0,0,30,100;3,3,100p,70p,0,0,30,100,12;4,2,375,100p,0,0,30,100;5,3,100p,70p,0,0,30,100;6,3,100p,70p,0,0,30,100;7,3,100p,70p,0,0,30,100;8,3,100p,70p,0,1,30,100](https://live.bilibili.com/p/html/live-app-red-envelope/popularitygiftlist.html?is_live_half_webview=1&hybrid_rotate_d=1&hybrid_biz=popularityRedPacket&is_cling_player=1&hybrid_half_ui=1,3,100p,70p,0,0,30,100,12;2,2,375,100p,0,0,30,100;3,3,100p,70p,0,0,30,100,12;4,2,375,100p,0,0,30,100;5,3,100p,70p,0,0,30,100;6,3,100p,70p,0,0,30,100;7,3,100p,70p,0,0,30,100;8,3,100p,70p,0,1,30,100)",
    "user_status": 2,
    "awards": [
      {
        "gift_id": 34758,
        "gift_name": "棒棒糖",
        "gift_pic": "[https://s1.hdslb.com/bfs/live/15313516b3ec0875d67130f18c0a53c582e76531.png](https://s1.hdslb.com/bfs/live/15313516b3ec0875d67130f18c0a53c582e76531.png)",
        "num": 5
      },
      {
        "gift_id": 34003,
        "gift_name": "人气票",
        "gift_pic": "[https://s1.hdslb.com/bfs/live/7164c955ec0ed7537491d189b821cc68f1bea20d.png](https://s1.hdslb.com/bfs/live/7164c955ec0ed7537491d189b821cc68f1bea20d.png)",
        "num": 5
      },
      {
        "gift_id": 31216,
        "gift_name": "小花花",
        "gift_pic": "[https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png](https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png)",
        "num": 5
      }
    ],
    "lot_config_id": 188,
    "total_price": 2000,
    "wait_num": 0,
    "wait_num_v2": 0,
    "is_mystery": false,
    "rp_type": 0,
    "sender_uinfo": {
      "uid": 26928797,
      "base": {
        "name": "mikufilck",
        "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
        "name_color": 0,
        "is_mystery": false,
        "risk_ctrl_info": null,
        "origin_info": {
          "name": "mikufilck",
          "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)"
        },
        "official_info": {
          "role": 0,
          "title": "",
          "desc": "",
          "type": -1
        },
        "name_color_str": ""
      },
      "medal": null,
      "wealth": null,
      "title": null,
      "guard": null,
      "uhead_frame": null,
      "guard_leader": null
    },
    "icon_url": "",
    "animation_icon_url": "",
    "rp_guard_info": null
  }
}
```
</details>
  

#### 直播间红包

当有人在直播间送出红包时触发，用于在公播界面展示“XXX送出红包”的消息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "POPULARITY_RED_POCKET_NEW"或"POPULARITY_RED_POCKET_V2_NEW" | 触发公屏展示送出红包提示语的消息,V2的内容和V1的内容完全一致，但是两个会被同时发送，可能是兼容新旧系统 |
| data | obj  | 发送者信息和红包（礼物）信息 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| lot_id | num | 红包唯一ID | |
| start_time | num | 开始抢红包的时间戳 | Unix时间戳，精确到秒 |
| current_time | num | 服务器当前时间戳 | Unix时间戳，精确到秒 |
| wait_num | num | 待调查 | |
| wait_num_v2 | num | 待调查 | |
| uname | str | 发送者名称 | |
| uid | num | 发送者UID | |
| action | str | 礼物操作 | 通常为"送出" |
| num | num | 礼物数量 | |
| gift_name | str | 礼物名称 | 通常为"红包" |
| gift_id | num | 礼物ID | 通常为13000 |
| price | num | 待调查 | 通常为红包对应的人民币/电池价值 |
| name_color | str | 发送者名称颜色 | |
| medal_info | obj | 发送者勋章信息 | 见下方展开 |
| wealth_level | num | 财富等级 | |
| group_medal | obj | 粉丝团信息 | 若无则为null |
| is_mystery | bool | 是否神秘人 | |
| sender_info | obj | 发送者详细资料 | 包含头像、认证等全套数据，见下方展开 |
| gift_icon | str | 红包图标 | 若无则为空字符串 |
| rp_type | num | 红包类型 | |

data的medal_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| target_id | num | 勋章归属者UID | |
| special | str | 特殊标识 | |
| icon_id | num | 图标ID | |
| anchor_uname | str | 主播昵称 | |
| anchor_roomid | num | 主播房间ID | |
| medal_level | num | 勋章等级 | |
| medal_name | str | 勋章名称 | |
| medal_color | num | 勋章颜色 | 十进制数值 |
| medal_color_start | num | 勋章渐变起始色 | 十进制数值 |
| medal_color_end | num | 勋章渐变结束色 | 十进制数值 |
| medal_color_border | num | 勋章边框颜色 | 十进制数值 |
| is_lighted | num | 是否点亮 | |
| guard_level | num | 舰队等级 | |

data的sender_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| uid | num | 发送者UID | |
| base | obj | 基础资料 | 见下方展开 |
| medal | obj | 勋章详情 | 若无则为null |
| wealth | obj | 财富信息 | 见下方展开 |
| title | obj | 称号信息 | 若无则为null |
| guard | obj | 舰队信息 | 见下方展开 |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| guard_leader | obj | 舰队队长状态 | 若无则为null |

data的sender_info的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 用户昵称 | |
| face | str | 头像链接 | |
| name_color | num | 昵称颜色 | |
| is_mystery | bool | 是否神秘人 | |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |
| origin_info | obj | 原始信息 | 见下方展开 |
| official_info | obj | 认证信息 | 见下方展开 |
| name_color_str | str | 昵称颜色字符串 | |

data的sender_info的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 原始昵称 | |
| face | str | 原始头像链接 | |

data的sender_info的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| role | num | 角色类型 | |
| title | str | 认证头衔 | |
| desc | str | 认证描述 | |
| type | num | 认证类型 | |

data的sender_info的wealth字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| level | num | 财富等级 | |
| dm_icon_key | str | 弹幕图标标识 | |

data的sender_info的guard字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| level | num | 舰队等级 | |
| expired_str | str | 过期时间字符串 | |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "POPULARITY_RED_POCKET_NEW",
  "data": {
    "lot_id": 27867696,
    "start_time": 1775820033,
    "current_time": 1775820033,
    "wait_num": 0,
    "wait_num_v2": 0,
    "uname": "mikufilck",
    "uid": 26928797,
    "action": "送出",
    "num": 1,
    "gift_name": "红包",
    "gift_id": 13000,
    "price": 20,
    "name_color": "",
    "medal_info": {
      "target_id": 0,
      "special": "",
      "icon_id": 0,
      "anchor_uname": "",
      "anchor_roomid": 0,
      "medal_level": 0,
      "medal_name": "",
      "medal_color": 0,
      "medal_color_start": 0,
      "medal_color_end": 0,
      "medal_color_border": 0,
      "is_lighted": 0,
      "guard_level": 0
    },
    "wealth_level": 33,
    "group_medal": null,
    "is_mystery": false,
    "sender_info": {
      "uid": 26928797,
      "base": {
        "name": "mikufilck",
        "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)",
        "name_color": 0,
        "is_mystery": false,
        "risk_ctrl_info": null,
        "origin_info": {
          "name": "mikufilck",
          "face": "[https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg](https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg)"
        },
        "official_info": {
          "role": 0,
          "title": "",
          "desc": "",
          "type": -1
        },
        "name_color_str": ""
      },
      "medal": null,
      "wealth": {
        "level": 33,
        "dm_icon_key": ""
      },
      "title": null,
      "guard": {
        "level": 0,
        "expired_str": ""
      },
      "uhead_frame": null,
      "guard_leader": null
    },
    "gift_icon": "",
    "rp_type": 0
  }
}
```
</details>

#### 直播间抢到红包的用户

当红包抽奖时间结束，开奖时接收到此消息，包含所有中奖用户及其获得的礼物信息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "POPULARITY_RED_POCKET_WINNER_LIST"或"POPULARITY_RED_POCKET_V2_WINNER_LIST" | 触发公屏展示红包开奖结果的消息。V2与V1结构基本一致，通常同时发送以兼容新旧系统。 |
| data | obj  | 抢到红包的用户信息、红包内的礼物信息 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| lot_id | num | 该红包的ID | |
| total_num | num | 该红包内所有礼物的总数 | |
| award_num | num | 发放的奖品数量 | **【注意】这是V1与V2的差异字段。** V1中包含实际数值，V2中通常归零为0。 |
| winner_info | array | 抢到红包的用户的信息数组 | 内部为数组形式，见下方展开 |
| awards | obj | 该红包内的礼物信息字典 | 见下方展开 |
| version | num | 版本号 | 通常为1 |
| rp_type | num | 红包类型 | |
| timestamp | num | 数据包发送的时间戳 | **【注意】这是V1与V2的差异字段。** V1中为精确到秒的Unix时间戳，V2中通常归零为0。 |

winner_info数组中的数组

| 索引 | 类型 | 内容 | 备注 |
| ---- | ---- | ---- | ---- |
| 0 | num | 用户的UID | 抢到该礼物的用户UID |
| 1 | str | 用户的名称 | 抢到该礼物的用户昵称 |
| 2 | num | 待调查 | |
| 3 | num | 抢到的礼物ID | 对应下文awards字段里的键名 |
| 4 | bool | 待调查 | 如false |
| 5 | null/obj | 待调查 | 目前观察到为null |
| 6 | num | 抢到礼物的时间戳 | Unix时间戳，精确到秒 |
| 7 | num | 主播UID | 当前红包所在直播间的主播UID |

awards字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| 礼物ID | obj | 礼物信息 | 键名为对应的礼物ID（如"34758"），值为该礼物的具体信息，见下方展开 |
| ... | obj | | |

礼物ID 对象

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| award_type | num | 待调查 | 通常为1 |
| award_name | str | 礼物的名称 | |
| award_pic | str | 礼物的图标URL | |
| award_big_pic | str | 礼物的高分辨率图标URL | |
| award_price | num | 礼物价值 | |

<details>
<summary>查看消息示例 (V1版)：</summary>

```json
{
  "cmd": "POPULARITY_RED_POCKET_WINNER_LIST",
  "data": {
    "lot_id": 27867696,
    "total_num": 15,
    "award_num": 13,
    "winner_info": [
      [
        353034971,
        "欧皇丹尼斯",
        14357970,
        34758,
        false,
        null,
        1775820335,
        1396521412
      ],
      [
        499498985,
        "桜を泣く",
        14610052,
        34003,
        false,
        null,
        1775820335,
        1396521412
      ],
      [
        12870323,
        "yuk608",
        14525734,
        31216,
        false,
        null,
        1775820335,
        1396521412
      ]
    ],
    "awards": {
      "34758": {
        "award_type": 1,
        "award_name": "棒棒糖",
        "award_pic": "[https://s1.hdslb.com/bfs/live/15313516b3ec0875d67130f18c0a53c582e76531.png](https://s1.hdslb.com/bfs/live/15313516b3ec0875d67130f18c0a53c582e76531.png)",
        "award_big_pic": "[https://i0.hdslb.com/bfs/live/a015e813040fb04f23ed5185baaaae124e410da5.png](https://i0.hdslb.com/bfs/live/a015e813040fb04f23ed5185baaaae124e410da5.png)",
        "award_price": 200
      },
      "34003": {
        "award_type": 1,
        "award_name": "人气票",
        "award_pic": "[https://s1.hdslb.com/bfs/live/7164c955ec0ed7537491d189b821cc68f1bea20d.png](https://s1.hdslb.com/bfs/live/7164c955ec0ed7537491d189b821cc68f1bea20d.png)",
        "award_big_pic": "[https://i0.hdslb.com/bfs/live/5bfaddf9a78e677501bb6d440f4d690668136496.png](https://i0.hdslb.com/bfs/live/5bfaddf9a78e677501bb6d440f4d690668136496.png)",
        "award_price": 100
      },
      "31216": {
        "award_type": 1,
        "award_name": "小花花",
        "award_pic": "[https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png](https://s1.hdslb.com/bfs/live/5126973892625f3a43a8290be6b625b5e54261a5.png)",
        "award_big_pic": "[https://i0.hdslb.com/bfs/live/cf90eac49ac0df5c26312f457e92edfff266f3f1.png](https://i0.hdslb.com/bfs/live/cf90eac49ac0df5c26312f457e92edfff266f3f1.png)",
        "award_price": 100
      }
    },
    "version": 1,
    "rp_type": 0,
    "timestamp": 1775820335
  }
}
```
</details>

#### 天选时刻开始

当主播在直播间发起“天选时刻”抽奖时接收到此消息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "ANCHOR_LOT_START" | 当天选时刻抽奖开始时为"ANCHOR_LOT_START" |
| data | obj  | 天选时刻的详细配置与奖品信息 | 见下方展开 |
| msg_id | str | 消息唯一ID | |
| p_is_ack | bool | 待调查 | |
| send_time | num | 发送时间戳 | 精确到毫秒 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| asset_icon | str | 资产图标URL | 通常为抽奖包或奖品的图片 |
| asset_icon_webp | str | webp格式图标URL | |
| award_content | str | 奖品内容 | 若无则为空字符串 |
| award_image | str | 奖品图片URL | |
| award_name | str | 奖品名称 | 例如"情书" |
| award_num | num | 奖品总数量 | |
| award_per_capita | num | 人均可获奖数 | 通常为1 |
| award_price_text | str | 奖品价值文本 | 例如"价值52电池" |
| award_type | num | 奖品类型 | 待调查（例如1可能代表虚拟礼物） |
| break_up_time | num | 打断时间 | 待调查 |
| cur_gift_num | num | 当前收到的礼物数 | 开启需要指定礼物的抽奖时有效 |
| current_time | num | 服务器当前时间戳 | 精确到秒 |
| danmu | str | 参与弹幕文本 | 用户参与抽奖时自动发送的弹幕内容 |
| danmu_new | array | 参与弹幕新结构 | 见下方展开 |
| danmu_type | num | 弹幕类型 | 待调查 |
| gift_id | num | 参与需赠送的礼物ID | 若为0则不需要送礼 |
| gift_name | str | 参与需赠送的礼物名 | |
| gift_num | num | 参与需赠送的礼物数量 | |
| gift_price | num | 礼物价格 | |
| goaway_time | num | UI消失时间 | 待调查 |
| goods_id | num | 物品ID | 常量如-99998 |
| icon_name | str | 图标名称 | 通常为"天选福袋" |
| id | num | 天选时刻唯一ID | 该场抽奖的唯一标识 |
| is_broadcast | num | 是否全站广播 | 待调查 |
| join_total | num | 参与总人数 | |
| join_type | num | 参与类型 | 待调查 |
| join_type_text | str | 参与类型文本 | |
| lot_status | num | 抽奖状态 | 通常0代表进行中 |
| max_time | num | 最大持续时间 | 单位为秒 |
| require_text | str | 参与条件文本 | 例如"关注主播" |
| require_type | num | 参与条件类型 | 例如1为关注 |
| require_value | num | 条件阈值 | |
| room_id | num | 房间ID | 发起抽奖的直播间真实ID |
| send_gift_ensure | num | 待调查 | |
| show_panel | num | 是否显示面板 | 1显示，0不显示 |
| sponsor_title | str | 赞助商头衔 | |
| start_dont_popup | num | 开始时不弹窗 | 待调查 |
| status | num | 当前状态 | |
| time | num | 剩余时间 | 单位为秒 |
| url | str | H5页面链接 | 包含UI参数的活动页面 |
| web_url | str | Web端页面链接 | 基础活动页面 |

data的danmu_new数组中的对象

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| danmu | str | 参与弹幕文本 | |
| danmu_view | str | 弹幕展示样式 | |
| reject | bool | 待调查 | 通常为false |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "ANCHOR_LOT_START",
  "data": {
    "asset_icon": "[https://i0.hdslb.com/bfs/live/cde7d1a68c0d10c6aa283c4c24b968578fa45d75.png](https://i0.hdslb.com/bfs/live/cde7d1a68c0d10c6aa283c4c24b968578fa45d75.png)",
    "asset_icon_webp": "[https://i0.hdslb.com/bfs/live/19b8a1b80f71af777ec615b329549224941b7b6c.webp](https://i0.hdslb.com/bfs/live/19b8a1b80f71af777ec615b329549224941b7b6c.webp)",
    "award_content": "",
    "award_image": "[https://s1.hdslb.com/bfs/live/14dafbf217618f0931c08897e0b3eefc00d0da22.png](https://s1.hdslb.com/bfs/live/14dafbf217618f0931c08897e0b3eefc00d0da22.png)",
    "award_name": "情书",
    "award_num": 1,
    "award_per_capita": 1,
    "award_price_text": "价值52电池",
    "award_type": 1,
    "break_up_time": 0,
    "cur_gift_num": 0,
    "current_time": 1775820389,
    "danmu": "加入粉丝团，参与天选时刻啦！",
    "danmu_new": [
      {
        "danmu": "加入粉丝团，参与天选时刻啦！",
        "danmu_view": "",
        "reject": false
      }
    ],
    "danmu_type": 0,
    "gift_id": 0,
    "gift_name": "",
    "gift_num": 0,
    "gift_price": 0,
    "goaway_time": 180,
    "goods_id": -99998,
    "icon_name": "天选福袋",
    "id": 14656751,
    "is_broadcast": 1,
    "join_total": 0,
    "join_type": 0,
    "join_type_text": "",
    "lot_status": 0,
    "max_time": 900,
    "require_text": "关注主播",
    "require_type": 1,
    "require_value": 0,
    "room_id": 1964690546,
    "send_gift_ensure": 0,
    "show_panel": 1,
    "sponsor_title": "",
    "start_dont_popup": 0,
    "status": 1,
    "time": 899,
    "url": "[https://live.bilibili.com/p/html/live-lottery/lottery-user.html?is_live_half_webview=1&hybrid_biz=live-lottery-anchor&hybrid_half_ui=1,3,100p,100p,0,0,30,0,0,1;2,2,375,100p,0,0,30,0,0,1;3,3,100p,100p,0,0,30,0,0,1;4,5,100p,100p,0,0,30,0,0,1;5,5,100p,100p,0,0,30,0,0,1;6,5,100p,100p,0,0,30,0,0,1;7,5,100p,100p,0,0,30,0,0,1;8,5,100p,100p,0,0,30,0,0,1](https://live.bilibili.com/p/html/live-lottery/lottery-user.html?is_live_half_webview=1&hybrid_biz=live-lottery-anchor&hybrid_half_ui=1,3,100p,100p,0,0,30,0,0,1;2,2,375,100p,0,0,30,0,0,1;3,3,100p,100p,0,0,30,0,0,1;4,5,100p,100p,0,0,30,0,0,1;5,5,100p,100p,0,0,30,0,0,1;6,5,100p,100p,0,0,30,0,0,1;7,5,100p,100p,0,0,30,0,0,1;8,5,100p,100p,0,0,30,0,0,1)",
    "web_url": "[https://live.bilibili.com/p/html/live-lottery/lottery-user.html](https://live.bilibili.com/p/html/live-lottery/lottery-user.html)"
  },
  "msg_id": "91505569478772224:1:1000",
  "p_is_ack": true,
  "send_time": 1775820391420
}
```
</details>

#### 天选时刻弹幕聚合

当天选时刻等活动期间，直播间短时间内出现大量相同的参与口令弹幕时，系统会专门发送此消息将口令折叠聚合以更新 UI 上的数量。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "DANMU_AGGREGATION" | 指示更新活动专属聚合弹幕的状态 |
| data | obj  | 聚合弹幕的具体信息和数量 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| activity_identity | str | 活动标识 | 对应具体活动的ID，例如天选时刻的 `id` |
| activity_source | num | 活动来源 | 待调查（通常为1） |
| aggregation_cycle | num | 聚合周期 | 待调查 |
| aggregation_icon | str | 聚合UI显示的图标 | 对应活动的图标链接 |
| aggregation_num | num | 聚合的弹幕数量 | 表示这段时间内有多少人发送了相同的这条弹幕口令 |
| broadcast_msg_type | num | 广播消息类型 | 待调查（通常为0） |
| msg | str | 被聚合的弹幕文本 | 例如"加入粉丝团，参与天选时刻啦！" |
| show_rows | num | 显示行数 | 聚合UI占据的行数 |
| show_time | num | 显示时间 | 待调查 |
| timestamp | num | 时间戳 | 服务器发送该数据包的Unix时间戳，精确到秒 |


<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "DANMU_AGGREGATION",
  "data": {
    "activity_identity": "14656751",
    "activity_source": 1,
    "aggregation_cycle": 1,
    "aggregation_icon": "[https://i0.hdslb.com/bfs/live/c8fbaa863bf9099c26b491d06f9efe0c20777721.png](https://i0.hdslb.com/bfs/live/c8fbaa863bf9099c26b491d06f9efe0c20777721.png)",
    "aggregation_num": 6,
    "broadcast_msg_type": 0,
    "msg": "加入粉丝团，参与天选时刻啦！",
    "show_rows": 1,
    "show_time": 2,
    "timestamp": 1775820481
  }
}
```
</details>

#### 天选时刻开奖

当天选时刻抽奖倒计时结束，公布中奖结果时接收到此消息。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "ANCHOR_LOT_AWARD" | 指示天选时刻已开奖 |
| data | obj  | 中奖结果及中奖用户名单信息 | 见下方展开 |
| msg_id | str | 消息唯一ID | |
| p_is_ack | bool | 待调查 | |
| send_time | num | 发送时间戳 | 精确到毫秒 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| award_dont_popup | num | 是否不弹窗 | 待调查（通常为1） |
| award_image | str | 奖品图片URL | |
| award_name | str | 奖品名称 | |
| award_num | num | 奖品总数量 | |
| award_per_capita | num | 人均可获奖数 | |
| award_price_text | str | 奖品价值文本 | 例如"价值52电池" |
| award_type | num | 奖品类型 | 待调查 |
| award_users | array | 中奖用户列表 | 包含所有抽中该奖品的用户信息，见下方展开 |
| id | num | 天选时刻唯一ID | 与发起抽奖时的ID一致 |
| lot_status | num | 抽奖状态 | 通常2代表已开奖/结束 |
| ruid | num | 发起抽奖的主播UID | |
| sponsor_title | str | 赞助商头衔 | |
| url | str | H5页面链接 | |
| web_url | str | Web端页面链接 | |

data的award_users数组中的对象

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| bag_id | num | 背包ID | 奖品发放到中奖用户背包的ID凭证 |
| color | num | 颜色 | 待调查 |
| face | str | 中奖者头像URL | |
| gift_id | num | 获得的奖品/礼物ID | |
| is_mystery | bool | 是否神秘人 | |
| level | num | 待调查 | |
| num | num | 中奖数量 | |
| uid | num | 中奖者UID | |
| uinfo | obj | 中奖者详细资料 | 包含头像、认证等全套数据，见下方展开 |
| uname | str | 中奖者昵称 | |

data的award_users的uinfo字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| uid | num | 用户UID | |
| base | obj | 基础资料 | 见下方展开 |
| medal | obj | 勋章详情 | 若无则为null |
| wealth | obj | 财富信息 | 若无则为null |
| title | obj | 称号信息 | 若无则为null |
| guard | obj | 舰队信息 | 若无则为null |
| uhead_frame | obj | 头像框信息 | 若无则为null |
| guard_leader | obj | 舰队队长状态 | 若无则为null |

data的award_users的uinfo的base字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 用户昵称 | |
| face | str | 头像链接 | |
| name_color | num | 昵称颜色 | |
| is_mystery | bool | 是否神秘人 | |
| risk_ctrl_info | obj | 风控信息 | 若无则为null |
| origin_info | obj | 原始信息 | 见下方展开 |
| official_info | obj | 认证信息 | 见下方展开 |
| name_color_str | str | 昵称颜色字符串 | |

data的award_users的uinfo的base的origin_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| name | str | 原始昵称 | |
| face | str | 原始头像链接 | |

data的award_users的uinfo的base的official_info字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| role | num | 角色类型 | |
| title | str | 认证头衔 | |
| desc | str | 认证描述 | |
| type | num | 认证类型 | |


<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "ANCHOR_LOT_AWARD",
  "data": {
    "award_dont_popup": 1,
    "award_image": "[https://s1.hdslb.com/bfs/live/14dafbf217618f0931c08897e0b3eefc00d0da22.png](https://s1.hdslb.com/bfs/live/14dafbf217618f0931c08897e0b3eefc00d0da22.png)",
    "award_name": "情书",
    "award_num": 1,
    "award_per_capita": 1,
    "award_price_text": "价值52电池",
    "award_type": 1,
    "award_users": [
      {
        "bag_id": 14519839,
        "color": 0,
        "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
        "gift_id": 31250,
        "is_mystery": false,
        "level": 0,
        "num": 1,
        "uid": 102581857,
        "uinfo": {
          "base": {
            "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
            "is_mystery": false,
            "name": "springtimes",
            "name_color": 0,
            "name_color_str": "",
            "official_info": {
              "desc": "",
              "role": 0,
              "title": "",
              "type": -1
            },
            "origin_info": {
              "face": "[https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg](https://i1.hdslb.com/bfs/face/291ec0601869615a6a3408d48549c61bba590e28.jpg)",
              "name": "springtimes"
            },
            "risk_ctrl_info": null
          },
          "guard": null,
          "guard_leader": null,
          "medal": null,
          "title": null,
          "uhead_frame": null,
          "uid": 102581857,
          "wealth": null
        },
        "uname": "springtimes"
      }
    ],
    "id": 14656751,
    "lot_status": 2,
    "ruid": 1396521412,
    "sponsor_title": "",
    "url": "[https://live.bilibili.com/p/html/live-lottery/lottery-user.html?is_live_half_webview=1&hybrid_biz=live-lottery-anchor&hybrid_half_ui=1,3,100p,100p,0,0,30,0,0,1;2,2,375,100p,0,0,30,0,0,1;3,3,100p,100p,0,0,30,0,0,1;4,5,100p,100p,0,0,30,0,0,1;5,5,100p,100p,0,0,30,0,0,1;6,5,100p,100p,0,0,30,0,0,1;7,5,100p,100p,0,0,30,0,0,1;8,5,100p,100p,0,0,30,0,0,1](https://live.bilibili.com/p/html/live-lottery/lottery-user.html?is_live_half_webview=1&hybrid_biz=live-lottery-anchor&hybrid_half_ui=1,3,100p,100p,0,0,30,0,0,1;2,2,375,100p,0,0,30,0,0,1;3,3,100p,100p,0,0,30,0,0,1;4,5,100p,100p,0,0,30,0,0,1;5,5,100p,100p,0,0,30,0,0,1;6,5,100p,100p,0,0,30,0,0,1;7,5,100p,100p,0,0,30,0,0,1;8,5,100p,100p,0,0,30,0,0,1)",
    "web_url": "[https://live.bilibili.com/p/html/live-lottery/lottery-user.html](https://live.bilibili.com/p/html/live-lottery/lottery-user.html)"
  },
  "msg_id": "91506511568756224:1:1000",
  "p_is_ack": true,
  "send_time": 1775821289867
}
```
</details>

#### 直播间看过人数

该数据包的正文中，前19字节的信息未知。

前19字节信息示例：
```
00000001: 8b38 8000 0000 7200 1000 0000 0000 0500  .8....r.........
00000002: 0000 00                                  ...
```
  
json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "WATCHED_CHANGE" | 若直播间看过人数更新，则内容是"WATCHED_CHANGE" |
| data | obj | 直播间看过人数 | |

data字段

|    字段    | 类型 |  内容  |    备注   |
| ---------- | --- | ------ | --------- |
| num        | num  | | |
| text_small | str | | |
| text_large | str | | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "WATCHED_CHANGE",
    "data": {
        "num": 17903,
        "text_small": "1.7万",
        "text_large": "1.7万人看过"
    }
}
```

</details>

#### 用户进场特效

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "ENTRY_EFFECT" | 带有进场特效的用户（如舰长、高等级用户）进入直播间时触发。 |
| data | obj | 进场用户、进场特效信息 | 包含了极具价值的 `uinfo` 底层结构。 |

data字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| id | num | 待调查 | |
| uid | num | 进场用户的UID | |
| target_id | num | 主播的UID | |
| mock_effect | num | 待调查 | |
| face | str | 进场用户的头像URL | |
| privilege_type | num | 待调查 | |
| copy_writing | str | 进场欢迎文本 | 注意：长名字可能会在此字段被截断并加上省略号。 |
| copy_color | str | 进场欢迎文本的十六进制颜色值 | |
| highlight_color | str | 待调查 | |
| priority | num | 待调查 | |
| basemap_url | str | 进场特效背景图片URL | APP端使用该URL |
| show_avatar | num | 是否显示用户头像 | 1显示<br/>0不显示 |
| web_basemap_url | str | 进场特效背景图片URL | 网页端使用该URL |
| web_effective_time | num | 进场特效生存时间 | 网页端 |
| web_effect_close | num | 待调查 | |
| web_close_time | num | 待调查 | |
| business | num | 待调查 | |
| copy_writing_v2 | str | 进场欢迎文本的完整版 | 通常包含完整的未经截断的用户名。 |
| icon_list | array | 待调查 | |
| max_delay_time | num | 待调查 | |
| trigger_time | num | 触发的Unix时间戳，以及后面9位未知数字 | |
| identities | num | 待调查 | |
| effect_silent_time | num | 待调查 | |
| effective_time_new | num | 待调查 | |
| web_dynamic_url_webp | str | 待调查 | |
| web_dynamic_url_apng | str | 待调查 | |
| mobile_dynamic_url_webp | str | 待调查 | |
| wealthy_info | obj | 用户财富等级信息 | |
| new_style | num | 待调查 | |
| is_mystery | bool | 是否神秘人 | |
| uinfo | obj | 进场用户的详细底层信息 | 核心数据结构，包含了精准的用户名和粉丝牌。 |
| full_cartoon_id | num | 待调查 | |
| priority_level | num | 待调查 | |
| wealth_style_info | obj | 财富等级特效样式 | |

data.uinfo字段 (核心用户信息)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| uid | num | 用户UID | |
| base | obj | 基础信息 | 包含准确的昵称和头像。 |
| medal | obj | 粉丝勋章信息 | 用户当前佩戴的粉丝牌数据。 |
| wealth | obj | 全站财富信息 | |
| title | str | 待调查 | 可能为null |
| guard | obj | 大航海(舰队)信息 | |
| uhead_frame | obj | 头像框信息 | 可能为null |
| guard_leader | obj | 待调查 | 可能为null |

data.uinfo.base字段

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| name | str | 进场用户的完整昵称 | 建议直接提取此字段，避免截断问题。 |
| face | str | 头像URL | |
| name_color | num | 名字颜色 | |
| is_mystery | bool | 是否隐藏身份 | |
| risk_ctrl_info | obj | 风控相关信息 | 可能为null |
| origin_info | obj | 待调查 | 可能为null |
| official_info | obj | 官方认证信息 | 可能为null |
| name_color_str | str | 名字颜色字符串 | |

data.uinfo.medal字段 (粉丝勋章)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| name | str | 粉丝勋章名称 | |
| level | num | 粉丝勋章等级 | |
| color_start | num | 背景渐变起始色 | |
| color_end | num | 背景渐变结束色 | |
| color_border | num | 边框颜色 | |
| color | num | 基础颜色 | |
| id | num | 粉丝牌ID | |
| typ | num | 待调查 | |
| is_light | num | 粉丝牌是否点亮 | 1点亮<br/>0熄灭 |
| ruid | num | 发放此粉丝牌的主播UID | |
| guard_level | num | 对应的大航海等级 | 0无<br/>1总督<br/>2提督<br/>3舰长 |
| score | num | 亲密度/积分 | |
| v2_medal_color_text | str | 粉丝牌文字颜色(V2) | Hex格式，例如 "#FFFFFF" |

data.uinfo.guard字段 (大航海/舰队)

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| level | num | 大航海等级 | 0无<br/>1总督<br/>2提督<br/>3舰长 |
| expired_str | str | 过期时间 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
  "cmd": "ENTRY_EFFECT",
  "data": {
    "id": 382,
    "uid": 26928797,
    "target_id": 3546384651258599,
    "mock_effect": 0,
    "face": "https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg",
    "privilege_type": 0,
    "copy_writing": "<%mikufil...%> 来了",
    "copy_color": "#F7F7F7",
    "highlight_color": "#FFFFFF",
    "priority": 1,
    "basemap_url": "",
    "show_avatar": 0,
    "effective_time": 0,
    "web_basemap_url": "https://i0.hdslb.com/bfs/live/mlive/5b14d1fc398f891c01f18e54a3384e75914d9333.png",
    "web_effective_time": 4,
    "web_effect_close": 1,
    "web_close_time": 900,
    "business": 6,
    "copy_writing_v2": "<%mikufilck%> 来了",
    "icon_list": [],
    "max_delay_time": 7,
    "trigger_time": 1774791005106451572,
    "identities": 1,
    "effect_silent_time": 0,
    "effective_time_new": 0,
    "web_dynamic_url_webp": "",
    "web_dynamic_url_apng": "",
    "mobile_dynamic_url_webp": "",
    "wealthy_info": {
      "uid": 0,
      "level": 32,
      "level_total_score": 0,
      "cur_score": 0,
      "upgrade_need_score": 0,
      "status": 0,
      "dm_icon_key": ""
    },
    "new_style": 1,
    "is_mystery": false,
    "uinfo": {
      "uid": 26928797,
      "base": {
        "name": "mikufilck",
        "face": "https://i2.hdslb.com/bfs/face/6a0dd36ba7fa18e84c6527c87beba02a5d2de3f9.jpg",
        "name_color": 0,
        "is_mystery": false,
        "risk_ctrl_info": null,
        "origin_info": null,
        "official_info": null,
        "name_color_str": ""
      },
      "medal": {
        "name": "做猫的",
        "level": 21,
        "color_start": 1725515,
        "color_end": 5414290,
        "color_border": 1725515,
        "color": 1725515,
        "id": 1228772,
        "typ": 0,
        "is_light": 1,
        "ruid": 3546384651258599,
        "guard_level": 0,
        "score": 2498,
        "guard_icon": "",
        "honor_icon": "",
        "v2_medal_color_start": "#3FB4F699",
        "v2_medal_color_end": "#3FB4F699",
        "v2_medal_color_border": "#3FB4F699",
        "v2_medal_color_text": "#FFFFFF",
        "v2_medal_color_level": "#3FB4F6E6",
        "user_receive_count": 0
      },
      "wealth": {
        "level": 32,
        "dm_icon_key": ""
      },
      "title": null,
      "guard": {
        "level": 0,
        "expired_str": ""
      },
      "uhead_frame": null,
      "guard_leader": null
    },
    "full_cartoon_id": 1800,
    "priority_level": 0,
    "wealth_style_info": {
      "url": "https://i0.hdslb.com/bfs/live/8dac191a14bdef6bb28cd2465b9f58bc3719e072.png"
    }
  }
}
```

</details>


#### 直播间在所属分区的排名改变

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "AREA_RANK_CHANGED" | 若直播间在所属分区的排名改变，则内容是"AREA_RANK_CHANGED" |
| data | obj | 直播间在所属分区的排名信息 | |

data字段

|    字段    | 类型 |  内容  |    备注   |
| ---------- | --- | ------ | --------- |
| conf_id | num | 待调查 | |
| rank_name | str | 排行榜名称 | |
| uid | num | 主播的UID | |
| rank | num | 直播间在分区的排名 | 若没有上榜则为0 |
| icon_url_blue | str | 蓝色排名图标URL | |
| icon_url_pink | str | 粉色排名图标URL | |
| icon_url_grey | str | 灰色排名图标URL | |
| action_type | num | 待调查 | |
| timestamp | num | 触发时的Unix时间戳 | |
| msg_id | str | 待调查 | |
| jump_url_link | str | 排行榜跳转链接 | APP端页面 |
| jump_url_pc | str | 排行榜跳转链接 | APP端页面 |
| jump_url_pink | str | 排行榜跳转链接 | APP端页面 |
| jump_url_web | str | 排行榜跳转链接 | APP端页面 |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "AREA_RANK_CHANGED",
    "data": {
        "conf_id": 23,
        "rank_name": "手游航海",
        "uid": 27717502,
        "rank": 4,
        "icon_url_blue": "https://i0.hdslb.com/bfs/live/18e2990a546d33368200f9058f3d9dbc4038eb5c.png",
        "icon_url_pink": "https://i0.hdslb.com/bfs/live/a6c490c36e88c7b191a04883a5ec15aed187a8f7.png",
        "icon_url_grey": "https://i0.hdslb.com/bfs/live/cb7444b1faf1d785df6265bfdc1fcfc993419b76.png",
        "action_type": 1,
        "timestamp": 1673625610,
        "msg_id": "e93c7860-b901-41ca-aad8-fe538a5fac9c",
        "jump_url_link": "https://live.bilibili.com/p/html/live-app-hotrank/index.html?clientType=3&ruid=27717502&conf_id=23&is_live_half_webview=1&hybrid_rotate_d=1&is_cling_player=1&hybrid_half_ui=1,3,100p,70p,f4eefa,0,30,100,0,0;2,2,375,100p,f4eefa,0,30,100,0,0;3,3,100p,70p,f4eefa,0,30,100,0,0;4,2,375,100p,f4eefa,0,30,100,0,0;5,3,100p,70p,f4eefa,0,30,100,0,0;6,3,100p,70p,f4eefa,0,30,100,0,0;7,3,100p,70p,f4eefa,0,30,100,0,0;8,3,100p,70p,f4eefa,0,30,100,0,0#/area-rank",
        "jump_url_pc": "https://live.bilibili.com/p/html/live-app-hotrank/index.html?clientType=4&ruid=27717502&conf_id=23&pc_ui=338,465,f4eefa,0#/area-rank",
        "jump_url_pink": "https://live.bilibili.com/p/html/live-app-hotrank/index.html?clientType=1&ruid=27717502&conf_id=23&is_live_half_webview=1&hybrid_rotate_d=1&is_cling_player=1&hybrid_half_ui=1,3,100p,70p,f4eefa,0,30,100,0,0;2,2,375,100p,f4eefa,0,30,100,0,0;3,3,100p,70p,f4eefa,0,30,100,0,0;4,2,375,100p,f4eefa,0,30,100,0,0;5,3,100p,70p,f4eefa,0,30,100,0,0;6,3,100p,70p,f4eefa,0,30,100,0,0;7,3,100p,70p,f4eefa,0,30,100,0,0;8,3,100p,70p,f4eefa,0,30,100,0,0#/area-rank",
        "jump_url_web": "https://live.bilibili.com/p/html/live-app-hotrank/index.html?clientType=2&ruid=27717502&conf_id=23#/area-rank"
    }
}
```

</details>


#### 直播间在所属分区排名提升的祝福

会分多个普通包发送

json格式

| 字段 | 类型 |   内容  |    备注   |
| ---- | ---- | ------ | --------- |
| cmd  | str | "COMMON_NOTICE_DANMAKU" | 例如提示“恭喜主播 时雨ioo 成为手游航海当前第5名”，<br/>，则内容是"COMMON_NOTICE_DANMAKU" |
| data | obj | 直播间在所属分区排名提升的祝福的信息 | |

data字段

|    字段    | 类型 |  内容  |    备注   |
| ---------- | --- | ------ | --------- |
| biz_id | num | 待调查 | |
| content_segments | array | 文本分段 | |
| danmaku_style | obj | 文本样式信息 | |
| danmaku_url | str | 待调查 | |
| dmscore | num | 待调查 | |
| terminals | array | 待调查 | |

content_segments数组中的对象

|    字段    | 类型 |  内容  |    备注   |
| ---------- | --- | ------ | --------- |
| font_color | str | text字段的十六进制颜色值 | |
| font_color_dark | str | text字段的十六进制颜色值 | APP端设置为深色模式时使用 |
| text | str | 祝贺文本 | |
| type | num | 待调查 | |

danmaku_style字段

|    字段    | 类型 |  内容  |    备注   |
| ---------- | --- | ------ | --------- |
| background_color | str | 文本背景颜色的十六进制颜色值 | |
| background_color_dark | str | 文本背景颜色的十六进制颜色值 | APP端设置为深色模式时使用 |

<details>
<summary>查看消息示例：</summary>

第一条数据：
```json
{
    "cmd": "COMMON_NOTICE_DANMAKU",
    "data": {
        "biz_id": 0,
        "content_segments": [
            {
                "font_color": "#CCCCCC",
                "font_color_dark": "#CCCCCC",
                "text": "恭喜主播 时雨ioo ",
                "type": 1
            },
            {
                "font_color": "#F494AF",
                "font_color_dark": "#F494AF",
                "text": "成为手游航海当前第5名",
                "type": 1
            }
        ],
        "danmaku_style": {
            "background_color": null,
            "background_color_dark": null
        },
        "danmaku_uri": "",
        "dmscore": 144,
        "terminals": [
            1,
            2,
            3
        ]
    }
}
```
第二条数据：
```json
{
    "cmd": "COMMON_NOTICE_DANMAKU",
    "data": {
        "biz_id": 0,
        "content_segments": [
            {
                "font_color": "#99A5AE",
                "font_color_dark": "#99A5AE",
                "text": "恭喜主播 时雨ioo 成为手游航海当前第5名",
                "type": 1
            }
        ],
        "danmaku_style": {
            "background_color": null,
            "background_color_dark": null
        },
        "danmaku_uri": "",
        "dmscore": 144,
        "terminals": [
            5
        ]
    }
}
```
第三条数据：
```json
{
    "cmd": "COMMON_NOTICE_DANMAKU",
    "data": {
        "biz_id": 0,
        "content_segments": [
            {
                "font_color": "#998EFF",
                "font_color_dark": "#998EFF",
                "text": "恭喜主播 时雨ioo 成为手游航海第5名",
                "type": 1
            }
        ],
        "danmaku_style": {
            "background_color": null,
            "background_color_dark": null
        },
        "danmaku_uri": "",
        "dmscore": 144,
        "terminals": [
            4
        ]
    }
}
```

</details>


#### 直播间信息更改

当主播手动修改直播间标题，或者更改直播分区时触发。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "ROOM_CHANGE" | 指示直播间标题或分区发生更改 |
| data | obj  | 变更后的标题、分区及场次信息 | 见下方展开 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| title | str | 直播间标题 | 变更后的新标题 |
| area_id | num | 子分区ID | 当前直播间所属二级分区的ID（如：371） |
| parent_area_id | num | 父分区ID | 当前直播间所属一级分区的ID（如：9） |
| area_name | str | 子分区名称 | 当前直播间所属二级分区的名称（如：虚拟主播） |
| parent_area_name | str | 父分区名称 | 当前直播间所属一级分区的名称（如：虚拟主播） |
| live_key | str | 直播场次密钥 | 当前直播的全局唯一标识符。如果主播在未开播时修改标题，此值通常为 `"0"` |
| sub_session_key | str | 子场次密钥 | 格式为 `[live_key]sub_time:[时间戳]`。直播中途切换分区时，时间戳部分会刷新 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "ROOM_CHANGE",
  "data": {
    "title": "开始白给CS",
    "area_id": 371,
    "parent_area_id": 9,
    "area_name": "虚拟主播",
    "parent_area_name": "虚拟主播",
    "live_key": "320830629635915849",
    "sub_session_key": "320830629635915849sub_time:1673690546"
  }
}
```
</details>


#### 醒目留言按钮状态

指示直播间“醒目留言（Super Chat）”功能的按钮是否开启及相关 UI 参数。
**【触发机制】** 该功能与直播间所属的“分区”强绑定。当主播开播，或者在直播中途将分区切换到支持 SC 的分区（如“虚拟主播”）时，系统会下发此指令，通知客户端渲染 SC 发送按钮。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "SUPER_CHAT_ENTRANCE" | 指示更新醒目留言按钮（入口）的状态 |
| data | obj  | 醒目留言按钮的具体信息 | 见下方展开 |
| roomid | str | 直播间ID | 注意：实际数据中为字符串格式。通常为真实房间ID |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| status | num | 开启状态 | 1 代表开启/显示醒目留言按钮，0 代表关闭/隐藏 |
| jump_url | str | 弹窗H5链接 | 观众点击“醒目留言”按钮后，客户端弹出的用于输入 SC 字符和付款的半屏 Webview 页面链接 |
| icon | str | 按钮图标URL | “醒目留言”按钮在 UI 上显示的图片链接 |
| broadcast_type | num | 广播类型 | 待调查（通常为1，可能表示下发给当前房间的所有用户） |

<details>
<summary>查看消息示例 (分区切换后开启 SC 入口)：</summary>

```json
{
  "cmd": "SUPER_CHAT_ENTRANCE",
  "data": {
    "status": 1,
    "jump_url": "[https://live.bilibili.com/p/html/live-app-superchat2/index.html?is_live_half_webview=1&hybrid_half_ui=1,3,100p,70p,ffffff,0,30,100;2,2,375,100p,ffffff,0,30,100;3,3,100p,70p,ffffff,0,30,100;4,2,375,100p,ffffff,0,30,100;5,3,100p,60p,ffffff,0,30,100;6,3,100p,60p,ffffff,0,30,100;7,3,100p,60p,ffffff,0,30,100](https://live.bilibili.com/p/html/live-app-superchat2/index.html?is_live_half_webview=1&hybrid_half_ui=1,3,100p,70p,ffffff,0,30,100;2,2,375,100p,ffffff,0,30,100;3,3,100p,70p,ffffff,0,30,100;4,2,375,100p,ffffff,0,30,100;5,3,100p,60p,ffffff,0,30,100;6,3,100p,60p,ffffff,0,30,100;7,3,100p,60p,ffffff,0,30,100)",
    "icon": "[https://i0.hdslb.com/bfs/live/0a9ebd72c76e9cbede9547386dd453475d4af6fe.png](https://i0.hdslb.com/bfs/live/0a9ebd72c76e9cbede9547386dd453475d4af6fe.png)",
    "broadcast_type": 1
  },
  "roomid": "3128551"
}
```
</details>

#### 互动游戏状态变更

当主播开启、关闭或变更开放平台互动玩法（如弹幕互动游戏、养成挂机游戏等）的状态时触发。这是一个轻量级的状态通知包，通常伴随着更详细的游戏配置大包一起下发。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "LIVE_INTERACT_GAME_STATE_CHANGE" | 指示互动游戏状态发生变更 |
| data | obj  | 状态详情 | 见下方展开 |
| recv_time | str | 接收时间 | 格式如 `"2026-04-16 22:20:31"` |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| game_name | str | 游戏大类名称 | 如 `"互动玩法"` |
| game_id | str | 游戏实例ID | 该局互动游戏的 UUID 标识符，如 `"b36a57c1-f772-42ec-9280-b6dcea29f15c"` |
| action | num | 动作状态码 | 如 `1` 可能代表开启/进行中 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "LIVE_INTERACT_GAME_STATE_CHANGE",
  "data": {
    "game_name": "互动玩法",
    "game_id": "b36a57c1-f772-42ec-9280-b6dcea29f15c",
    "action": 1
  },
  "recv_time": "2026-04-16 22:20:31"
}
```

</details>

#### 开放平台互动游戏配置大包

当主播开启“开放平台互动游戏”（如弹幕修仙、云养猫等互动玩法）时，服务器下发的核心数据包。它包含了极其庞大的游戏规则说明、弹幕触发词表、关联的专属互动礼物列表，以及供前端（客户端/Web端）渲染 iframe 互动面板的 UI 坐标配置。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "LIVE_OPEN_PLATFORM_GAME" | 开放平台互动游戏全局配置指令 |
| data | obj  | 游戏详细配置 | 包含实例ID、序列化的游戏指令集与UI面板配置 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| msg_type | str | 消息类型 | 如 `"game_start"` 代表游戏开始。 |
| msg_sub_type | str | 消息子类型 | 通常与 `msg_type` 保持一致。 |
| game_name | str | 游戏名称 | 真实的互动游戏名称，如 `"猫猫养成Plus"`。 |
| game_code | str | 游戏代码 | 游戏的内部数字标号（如 `"3548430620272881"`）。 |
| game_id | str | 游戏实例ID | 本局游戏的唯一 UUID，与 `LIVE_INTERACT_GAME_STATE_CHANGE` 对应。 |
| game_msg | str | **序列化的游戏信息** | **【核心字段】** 包含弹幕指令表、专属礼物表和长文本说明的 JSON 字符串。需在代码中进行二次 `JSON.parse`。见下方展开。 |
| interactive_panel_conf | str | **序列化的UI面板配置** | 包含前端渲染该玩法小窗的坐标、缩放参数和 Webview URL 链接的 JSON 字符串。 |
| timestamp | num | 时间戳 | 秒级。 |
| block_uids | array | 黑名单/屏蔽列表 | 通常为空数组。 |
| panel_status | num | 面板状态 | 如 `0`。 |

data.game_msg (需反序列化的核心业务大包)

> **开发者注**：将 `game_msg` 字符串解析为 JSON 对象后，将得到以下结构。这是互动玩法弹幕机和数据统计最需要的核心数据。

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| play_name | str | 玩法名称 | 同外层 `game_name`。 |
| command_instructions | str | 弹幕指令说明文案 | 极长的一段文本，包含教观众发什么弹幕触发什么功能的说明（自带换行符 `\n`）。 |
| gift_instructions | str | 礼物说明文案 | 说明哪些礼物能产生什么效果的文本。 |
| play_instructions | str | 玩法简介 | 一句话简介。 |
| dm_command | array | **弹幕触发词表** | 包含用户可发送的弹幕关键字及其效果映射。见下方展开。 |
| gift_command | array | **专属互动礼物表** | 包含该游戏关联的所有特殊礼物的 ID、名称、价格和限制。见下方展开。 |
| enable_custom_status | num | 自定义状态开启标识 | |

dm_command 数组 (弹幕触发词映射)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| dm_text | str | 弹幕触发词 | 观众需要发送的具体文本（如 `"喵"`, `"左"`, `"召唤喵娘"`）。 |
| dm_key | str | 动作键值 | 后端/游戏引擎识别的按键动作指令（如 `"左1"`）。 |
| dm_effect | str | 触发效果说明 | 用于在说明界面展示此弹幕的具体作用（如 `"往左移动1"`）。 |

gift_command 数组 (专属互动礼物配置)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_id | num | 礼物ID | 唯一的礼物标识（与 `GIFT_PANEL_PLAN` 中的一致）。 |
| gift_name | str | 礼物名称 | 如 `"毛线球"`, `"逗猫棒"`。 |
| gift_desc | str | 礼物效果描述 | 包含具体增加多少属性的说明文案。 |
| gift_icon | str | 静态图标URL | 礼物的展示图片。 |
| gift_price_gold | num | 礼物价格(内部) | 通常需除以 1000（如 `98000` 实际上是 98 电池）。 |
| gift_price_cell | num | 礼物价格(前端) | 实际的电池价值（如 `980`）。 |
| max_send_limit | num | 最大单次连送限制 | |
| batch_map | array | 连送选项菜单 | 用户点击赠送时弹出的连送阶梯（如 `[10, 100, 520]`）。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "LIVE_OPEN_PLATFORM_GAME",
  "data": {
    "msg_type": "game_start",
    "game_name": "猫猫养成Plus",
    "game_id": "b36a57c1-f772-42ec-9280-b6dcea29f15c",
    "game_msg": "{\"play_name\":\"猫猫养成Plus\",\"command_instructions\":\"猫猫养成玩法介绍：\\n...\",\"dm_command\":[{\"dm_text\":\"喵\",\"dm_key\":\"喵\",\"dm_effect\":\"显示修炼特效\"}],\"gift_command\":[{\"gift_id\":34492,\"gift_desc\":\"直播间累计赠送200个触发特效！\",\"gift_name\":\"蝶恋花\",\"gift_price_gold\":100}]}",
    "interactive_panel_conf": "{\"dragHeight\":50,\"dragWidth\":100,\"iframeHeight\":316,\"iframeWidth\":375,\"url\":\"//[live.bilibili.com/blackboard/activity-rjZIC2Uwy6.html](https://live.bilibili.com/blackboard/activity-rjZIC2Uwy6.html)\"}",
    "timestamp": 1776349231
  }
}
```

</details>

#### 互动游戏按钮状态变更

当互动游戏开启或关闭时，服务器下发此指令以控制客户端界面中“互动玩法”专属按钮（通常为一个“小手柄”图标）的显示状态和点击响应行为。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "OPENPLATFORM_GAME_BUTTON_STATUS_CHANGE" | 指示更新互动游戏按钮状态 |
| data | obj  | 按钮详情 | 包含跳转链接与状态码 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| status | num | 按钮状态 | `1` 通常代表开启/显示，`0` 代表关闭/隐藏。 |
| jump_url | str | 交互面板地址 | 玩家点击“小手柄”按钮后弹出的半屏 Webview / iframe 页面链接。URL 中通常携带 `hybrid_half_ui` 等复杂的客户端渲染参数。 |

<details>
<summary>查看消息示例：</summary>

```json
{
  "cmd": "OPENPLATFORM_GAME_BUTTON_STATUS_CHANGE",
  "data": {
    "jump_url": "[https://live.bilibili.com/blackboard/activity-rjZIC2Uwy6.html?is_live_half_webview=1&hybrid_half_ui=1,3,100p,370,0,0,0,0,0,1;2,2,375,100p,0,0,0,0,0,1](https://live.bilibili.com/blackboard/activity-rjZIC2Uwy6.html?is_live_half_webview=1&hybrid_half_ui=1,3,100p,370,0,0,0,0,0,1;2,2,375,100p,0,0,0,0,0,1)...",
    "status": 1
  }
}
```

</details>

#### 直播间控制面板动态配置

服务器向客户端推送的播放器面板功能按钮配置指令。通常在进入直播间，或主播临时开关特定功能（如语音连麦、小窗播放、弹幕设置等）时触发，用于动态控制客户端底部的按钮入口及其对应图标和跳转链接。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "LIVE_PANEL_CHANGE_CONTENT" | 指示更新直播间控制面板按钮配置 |
| data | obj  | 布局大包 | 包含内部设置项和外部独立悬浮按钮。 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| setting_list | array | 内部设置列表 | **隐藏在“设置/更多”菜单中的功能**（如画质、小窗播放、投屏、举报等）。 |
| outer_list | array | 外部按钮列表 | **直接展示在播放器底部或侧边的核心按钮**（如送礼、购物车、语音连麦）。 |
| interaction_list | array | 互动挂件列表 | 悬浮在画面边缘的特殊入口（如装扮中心、特殊活动入口）。 |

> 💡 **开发者注**：以上三个数组内部包含的“按钮对象”结构完全一致。本列表剔除了大量客户端冗余的空白静态资源 URL（如 `panel_icon`, `dynamic_icon` 等），仅保留业务强相关的核心属性。

按钮对象 (Button Object) 核心结构

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| biz_id | num | 业务ID | **核心标识符**。固定代表某个功能（例如 `1012`为管理员, `33`为购物车, `2`为语音连麦）。 |
| title | str | 按钮标题 | 前端展示的主标题（如 `"语音连麦"`）。 |
| note | str | 按钮提示 | 附加文案说明（如 `"连麦功能关闭"` 或 `"等待中"`）。 |
| icon | str | 静态图标 | 该按钮的基础 PNG 图标。 |
| jump_url | str | 跳转链接 | 如果此按钮触发页面跳转，则包含目标 URL（通常携带各类半屏 H5 的启动参数）。为空则代表客户端原生动作。 |
| weight | num | 排序权重 | 决定按钮的排列顺序，数字越大通常排在越前面。 |
| status_type | num | 状态分类 | 待调查。 |
| type_id | num | 分组ID | 如 `1` 为常规设置，`2` 为外置交互。 |
| custom | array/null | **动态状态配置** | **极其关键**！当按钮有多种状态时（例如连麦功能分为：关闭、可连麦、等待中、连麦中），此数组会定义每个状态对应的专属 `icon`、`note` 和 `status` 码。 |

<details>
<summary>查看消息示例 (包含连麦动态状态)：</summary>

```json
{
  "cmd": "LIVE_PANEL_CHANGE_CONTENT",
  "data": {
    "setting_list": [
      {
        "biz_id": 1012,
        "title": "管理员",
        "note": "管理员",
        "jump_url": "[https://live.bilibili.com/p/html/live-app-room-admin/index.html?is_live_half_webview=1#/roomManagement](https://live.bilibili.com/p/html/live-app-room-admin/index.html?is_live_half_webview=1#/roomManagement)"
      }
    ],
    "outer_list": [
      {
        "biz_id": 2,
        "title": "语音连麦",
        "note": " ",
        "weight": 94,
        "custom": [
          {
            "status": 2,
            "note": "连麦功能关闭",
            "icon": "[https://i0.hdslb.com/bfs/live/e3a8c212bc493b88a33fe1853a16270e22d9a70b.png](https://i0.hdslb.com/bfs/live/e3a8c212bc493b88a33fe1853a16270e22d9a70b.png)"
          },
          {
            "status": 1,
            "note": "连麦",
            "icon": "[https://i0.hdslb.com/bfs/live/b8cabd73def53d85bd092f4e8b3f9f6534ec2dc6.png](https://i0.hdslb.com/bfs/live/b8cabd73def53d85bd092f4e8b3f9f6534ec2dc6.png)"
          },
          {
            "status": 3,
            "note": "等待中",
            "icon": "[https://i0.hdslb.com/bfs/live/c25451d846c5c36a56874626c6496743e6c8b726.webp](https://i0.hdslb.com/bfs/live/c25451d846c5c36a56874626c6496743e6c8b726.webp)"
          },
          {
            "status": 4,
            "note": "连麦中",
            "icon": "[https://i0.hdslb.com/bfs/live/bcf5f48883ddbb96c8680bcc9ed2d4c11798e526.webp](https://i0.hdslb.com/bfs/live/bcf5f48883ddbb96c8680bcc9ed2d4c11798e526.webp)"
          }
        ]
      }
    ]
  }
}
```

</details>

#### 礼物面板动态配置

服务器向客户端推送的礼物面板更新指令。通常在进入直播间，或主播开启特定互动玩法时触发，用于动态更新礼物栏中的特定礼物列表、价格、角标（如“玩法”、“专属”）以及基础动效资源。

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd  | str  | "GIFT_PANEL_PLAN" | 指示更新礼物面板配置 |
| data | obj  | 配置详情 | 包含礼物列表和面板排序参数 |

data字段

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_list | array | 礼物配置列表 | **核心数据**。包含该面板下所有新增/更新的礼物详细属性。 |
| special_type_sort | array | 特殊类型排序 | 包含一组数字（如 `[12, 6, 14]`），控制礼物在面板中的显示顺序。 |
| action | num | 操作类型 | 如 `1` 代表更新操作。 |
| tab_id | num | 礼物栏分类ID | 指示这些礼物属于哪个特定的 Tab 页（如 `11` 可能是特定的玩法 Tab）。 |

data.gift_list数组 (礼物对象)

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| gift_id | num | 礼物ID | 唯一的礼物数字标识。 |
| config | obj | 礼物详细配置 | **核心业务属性**。见下方展开。 |
| special_type | num | 特殊礼物类型 | 与外层的 `special_type_sort` 对应，决定其分类或排序。 |
| show | bool | 是否显示 | `true` 为在面板中可见。 |

data.gift_list[x].config字段 (礼物核心业务配置)

> **开发者注**：原始 JSON 包中此对象极其庞大（含海量 `svga` 动画文件路径、渲染停留时长 `stay_time` 及内部商城 `goods_id` 等）。为了方便第三方开发者（如数据统计、弹幕机开发）快速接入，本表格已剔除客户端底层渲染噪音，**仅保留最具业务价值的核心字段**。

| 字段 | 类型 | 内容   |   备注   |
| ---- | ---- | ------ | -------- |
| id | num | 礼物ID | 同外层 `gift_id`。 |
| name | str | 礼物名称 | 如 `"逗猫棒"`, `"爱的魔法"`。 |
| price | num | 礼物价格 | 内部数值（通常需要除以 1000 转换为实际的电池/金瓜子数量）。 |
| coin_type | str | 货币类型 | 通常为 `"gold"` (金瓜子/电池)。 |
| desc | str | 礼物描述 | 包含触发特效的提示（如 `"直播间累计赠送200个触发特效！"`）。 |
| corner_mark | str | 面板角标文字 | 如 `"玩法"`，显示在礼物图标右上角。 |
| count_map | array | 连送选项列表 | 定义用户点击赠送时弹出的批量选项（如 `[{"num": 1}, {"num": 10}, {"num": 520}]`）。 |
| max_send_limit | num | 最大赠送限制 | 一次性最多能送出的数量。 |
| img_basic | str | 静态图标URL | 礼物的基础静态 PNG 图片。 |
| gif / webp | str | 动态图标URL | 礼物面板中播放的动态缩略图。 |
| web_light / web_dark | obj | 角标颜色配置 | 针对网页端明暗主题的角标背景颜色设定（包含 `corner_color_bg` 等）。 |

<details>
<summary>查看消息示例 (精简版)：</summary>

```json
{
  "cmd": "GIFT_PANEL_PLAN",
  "data": {
    "action": 1,
    "tab_id": 11,
    "special_type_sort": [12, 6, 14, 13, 5, 7, 8, 9],
    "gift_list": [
      {
        "gift_id": 34494,
        "show": true,
        "special_type": 6,
        "config": {
          "id": 34494,
          "name": "逗猫棒",
          "price": 98000,
          "coin_type": "gold",
          "corner_mark": "玩法",
          "desc": "喵！这是逗猫棒哎！猫猫超超超喜欢！来和猫猫一起动次打次~",
          "img_basic": "[https://s1.hdslb.com/bfs/open-live/195eba4b97b6a2c4da871b147c5066734fff214c.png](https://s1.hdslb.com/bfs/open-live/195eba4b97b6a2c4da871b147c5066734fff214c.png)",
          "gif": "[https://i0.hdslb.com/bfs/open-live/2df934703b29dcb9a7dc92a516e82b3522a6f281.gif](https://i0.hdslb.com/bfs/open-live/2df934703b29dcb9a7dc92a516e82b3522a6f281.gif)",
          "max_send_limit": 2040,
          "count_map": [
            { "num": 1 },
            { "num": 10 },
            { "num": 520 }
          ]
        }
      }
    ]
  }
}
```

</details>

#### 顶部横幅

网页端在直播间标题下面的横幅

例如“限时任务”等

json格式


| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "WIDGET_BANNER" | |
| data | obj | 横幅信息 | |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| timestamp | num | 服务器发送数据包时的Unix时间戳 | |
| widget_list | obj | 横幅信息 | 待调查 |

widget_list字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| 横幅ID | obj | 横幅信息 | |
| ... | obj | | |

横幅ID 字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| id | num | 横幅ID | |
| title | str | 待调查 | |
| cover | str | 待调查 | |
| web_cover | str | 待调查 | |
| tip_text | str | 待调查 | |
| tip_text_color | str | 待调查 | |
| tip_bottom_color | str | 待调查 | |
| jump_url | str | 点击横幅后出现小窗的页面的URL | |
| url | str | 待调查 | |
| stay_time | num | 待调查 | |
| site | num | 待调查 | |
| platform_in | array | 待调查 | |
| type | num | 待调查 | |
| band_id | num | 待调查 | |
| sub_key | str | 待调查 | |
| sub_data | str | 横幅数据 | |
| is_add | bool | 待调查 | |

<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "WIDGET_BANNER",
    "data": {
        "timestamp": 1673684868,
        "widget_list": {
            "308": {
                "id": 308,
                "title": "一月限时任务",
                "cover": "",
                "web_cover": "",
                "tip_text": "限时任务",
                "tip_text_color": "",
                "tip_bottom_color": "",
                "jump_url": "https://live.bilibili.com/activity/live-activity-battle/index.html?app_name=time_limited_task_jan_2023&is_live_half_webview=1&hybrid_rotate_d=1&hybrid_half_ui=1,3,100p,70p,0,0,0,0,12,0;2,2,375,100p,0,0,0,0,12,0;3,3,100p,70p,0,0,0,0,12,0;4,2,375,100p,0,0,0,0,12,0;5,3,100p,70p,0,0,0,0,12,0;6,3,100p,70p,0,0,0,0,12,0;7,3,100p,70p,0,0,0,0,12,0;8,3,100p,70p,0,0,0,0,12,0&room_id=8618057&uid=29857468#/",
                "url": "",
                "stay_time": 5,
                "site": 1,
                "platform_in": [
                    "live",
                    "blink",
                    "live_link",
                    "web",
                    "pc_link"
                ],
                "type": 1,
                "band_id": 101558,
                "sub_key": "",
                "sub_data": "%7B%22task_status%22%3A0%2C%22current_val%22%3A10%2C%22target_val%22%3A1200%2C%22timeout%22%3A1673687024%2C%22reward_price%22%3A8%2C%22reward_type%22%3A1%7D",
                "is_add": true
            }
        }
    }
}
```

</details>

#### 下播的直播间

json格式


| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "STOP_LIVE_ROOM_LIST" | |
| data | obj | 下播的直播间ID列表 | |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| room_id_list | array | 下播的直播间ID | |

room_id_list数组中的数字

| 类型 | 内容 | 备注 |
| --- | ---- | ---- |
| num | 下播的直播间ID | 未知是真实ID还是短号 |
| num | ... | |


<details>
<summary>查看消息示例：</summary>
  
```json
{
    "cmd": "STOP_LIVE_ROOM_LIST",
    "data": {
        "room_id_list": [
            22629205,
            23130005,
            25963791,
            5532805,
            668631,
            21409011,
            21559541,
            23499952,
            26700301,
            26785971,
            11673798,
            13766041,
            22980849,
            23719726,
            23865141,
            24984476,
            6134501,
            13782552,
            22276717,
            24107587,
            25023546,
            25404621,
            25516925,
            26527626,
            3392341,
            34027,
            502153,
            6479194,
            7636554,
            12237172,
            22821330,
            24484883,
            25641623,
            26230536,
            26792222,
            3642143,
            21774100,
            22797418,
            23698420,
            24020165,
            23969235,
            24207417,
            24541492,
            24900566,
            25385044,
            4484938,
            11113452,
            21442530,
            22046176,
            22184897,
            22386835,
            23499007,
            26129631,
            26866037,
            5971876,
            22779750,
            24132482,
            25789722,
            26251362,
            26822052,
            26835655,
            5122088,
            6668191,
            12439052,
            23690850,
            24458365,
            26189089,
            26676322,
            26872742,
            4917898,
            826723,
            22886872,
            24752347,
            25108137,
            5796786,
            6176498,
            6208022,
            7578115,
            14218725,
            22659435,
            23774701,
            24804876,
            25081572,
            25275744,
            26430916,
            730392,
            9505076,
            25467274,
            3015372,
            5764087,
            9407015,
            21356836,
            24302940,
            25469360,
            25666252,
            26564899,
            26574306,
            9391864,
            136707,
            15163029,
            22001560,
            22642183,
            24168773,
            24197349,
            26750190,
            59670,
            6545138,
            7538431,
            12568128,
            22865116,
            26566675,
            26658222,
            26778289,
            26856746,
            3386215,
            1270737,
            1856866,
            22371951,
            22953580,
            23026533,
            9316759,
            13628231,
            25166176,
            6736476,
            7745491,
            893989,
            25349228,
            25684996,
            26835833,
            763132,
            1282353,
            14333573,
            26677056,
            5553188,
            1549629,
            22807502,
            25633167,
            26062956,
            26558451,
            9312947,
            14366742,
            1864809,
            25581444,
            26656406,
            11454847,
            13507879,
            187331,
            22626880,
            23187177,
            23481929,
            24042533,
            24501754,
            26776408,
            2315619,
            24320832,
            24708829,
            26236176,
            26575516,
            3105045,
            6164089,
            21145740,
            21258252,
            23211964,
            23610573,
            26873451,
            10452273,
            21300836,
            26076163,
            26510266,
            933508,
            21751571,
            24043374,
            26045578,
            26784723,
            26811618,
            22836140,
            23558501,
            24429614,
            24476599,
            2681976,
            26867816,
            7802886,
            13617926,
            2049112,
            26233820,
            6868338,
            23458654,
            24370731,
            26126954,
            5070119,
            24416075
        ]
    }
}
```
  
</details>
  
#### 未知消息

`PLAY_TOGETHER`  
<details>
<summary>查看消息示例：</summary>

示例1：
```json
{
    "cmd": "PLAY_TOGETHER",
    "data": {
        "ruid": 29857468,
        "roomid": 8618057,
        "action": "switch_off",
        "uid": 0,
        "timestamp": 1673690546,
        "message": "",
        "message_type": 0,
        "jump_url": "",
        "web_url": "",
        "apply_number": 0,
        "refresh_tool": false,
        "cur_fleet_num": 0,
        "max_fleet_num": 0
    }
}
```
示例2
```json
{
    "cmd": "PLAY_TOGETHER",
    "data": {
        "ruid": 29857468,
        "roomid": 8618057,
        "action": "switch_off",
        "uid": 0,
        "timestamp": 1673690549,
        "message": "系统提示：主播已切换分区",
        "message_type": 3,
        "jump_url": "",
        "web_url": "",
        "apply_number": 0,
        "refresh_tool": true,
        "cur_fleet_num": 0,
        "max_fleet_num": 0
    }
}
```
</details>
  
  
