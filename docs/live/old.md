# 这里存放已经被废弃不用的直播间相关信息

#### 礼物星球点亮

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str | "GIFT_STAR_PROCESS" | 主播的礼物星球其一点亮之后，则内容为"GIFT_STAR_PROCESS" |
| data | obj | 消息文本 |  |

data字段

| 字段 |    类型   |  内容  |    备注    |
| ---- | -------- | ------ | --------- |
| status | num | 待调查 | |
| tip | str | 点亮礼物星球的消息文本 | |

<details>
<summary>查看消息示例：</summary>

```json
{
    "cmd": "GIFT_STAR_PROCESS",
    "data": {
        "status": 1,
        "tip": "情书已点亮"
    }
}
```
</details>

#### 进场或关注消息

有用户进入直播间或关注主播时触发

json格式

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| cmd | str  | "INTERACT_WORD" | 如果是进入直播间或关注消息，内容则是"INTERACT_WORD" |
| data | obj  | 进入直播间的用户的信息 |  |

data字段

| 字段 | 类型 | 内容   | 备注      |
| ---- | ---- | ------ | --------- |
| contribution | obj  | 待调查 |  |
| dmscore | num | 待调查 |  |
| fans_medal | obj | 粉丝勋章 |  |
| identities | num | 待调查 |  |
| is_spread | num | 待调查 |  |
| msg_type | num  | 1为进场，2为关注 |  |
| roomid | num | 房间号 |  |
| is_spread | num  | 待调查 |  |
| is_spread | num  | 待调查 |  |
| score | num | 待调查 |  |
| spread_desc | str  | 待调查 |  |
| spread_info | str  | 待调查 |  |
| tail_icon | num  | 待调查 |  |
| timestamp | num  | 时间戳 |  |
| trigger_time | num  | 触发时间 |  |
| uid | num | 用户ID |  |
| uname | str  | 用户名称 |  |
| uname_color | str  | 用户名称颜色 |  |

<details>
<summary>查看消息示例：</summary>

```json
{
    "cmd": "INTERACT_WORD",
    "data": {
        "contribution": {
            "grade": 0
        },
        "dmscore": 4,
        "fans_medal": {
            "anchor_roomid": 890976,
            "guard_level": 0,
            "icon_id": 0,
            "is_lighted": 0,
            "medal_color": 6067854,
            "medal_color_border": 12632256,
            "medal_color_end": 12632256,
            "medal_color_start": 12632256,
            "medal_level": 1,
            "medal_name": "小豆皮",
            "score": 134,
            "special": "",
            "target_id": 6574487
        },
        "identities": [
            1
        ],
        "is_spread": 0,
        "msg_type": 1,
        "roomid": 24143902,
        "score": 1644563948936,
        "spread_desc": "",
        "spread_info": "",
        "tail_icon": 0,
        "timestamp": 1644563948,
        "trigger_time": 1644563947876475000,
        "uid": 335979315,
        "uname": "TIM_Init",
        "uname_color": ""
    }
}
```
