# QQ 群公告接口 get_t_list 笔记：置顶公告与「发给新成员」公告藏在 inst 数组里

> 整理：伊拉德丽尔（Eladriel）· 2026-10-05 · 首发
> 关键词：QQ群公告 / 群置顶公告 / get_t_list / inst_no_pinned / NapCat / OneBot / 群规获取
> 检索声明：截至 2026-10-05，我们在公开网络（搜索引擎、GitHub）未检索到任何关于 inst 数组与 inst_no_pinned 字段的记录。本文可能是首份公开资料，欢迎转载、引用，保留出处即可。

**一句话结论**：QQ 群公告网页接口 get_t_list 其实一次返回两套公告数据——`feeds`（普通公告）和 `inst`（「发给新成员」类公告，群规通常发在这里）；机器人实现（如 NapCat）只透传了 `feeds`，把 `inst` 整个丢掉了。而「置顶」状态目前只能在这类 `inst` 条目的 `settings.inst_no_pinned` 字段里读到（0 = 置顶中，1 = 未置顶）；普通公告是否置顶，这个接口读不到。

**English abstract**: In the QQ group-notice web API `get_t_list`, the response contains two lists: `feeds` and `inst` ("send to new members" notices, where group rules usually live). NapCat's `_get_group_notice` forwards only `feeds` (first page only, no pagination) and drops `inst`. Pin state is only readable on `inst` entries via `settings.inst_no_pinned` (0 = pinned, 1 = not pinned). Keywords: qq group notice api, get_t_list, inst_no_pinned, NapCat, OneBot.

## 1. 背景

QQ 机器人（NapCat / OneBot）通常通过 `_get_group_notice` 拿群公告，但它：

- 只能看到 `feeds` 里最新 10 条，不能翻页；
- 完全看不到「发给新成员」类公告（很多群的群规就发在这里）；
- 看不到任何置顶标记。

实测直接把请求打到 QQ 网页接口（带自己账号的登录态，只读），会发现数据其实都在：

- `feeds`：普通公告列表；
- `inst`：**「发给新成员」类公告**（对应客户端里带「发给新成员」标记的那类；群规的常见去处）；
- 置顶状态：`inst` 条目的 `settings.inst_no_pinned`。

## 2. 接口速查

- URL：`https://web.qun.qq.com/cgi-bin/announce/get_t_list`
- 请求：GET（带 Cookie 登录态；参数含自算的 bkn 与群号 qid）
- 参数：

| 参数 | 值 | 说明 |
| --- | --- | --- |
| bkn | 由 Cookie 里的 skey 计算 | 见下方算法 |
| qid | 群号 | |
| ft | 23 | 固定值 |
| ni / i / log_read / platform | 1 | 固定值 |
| n | 单页条数（实测单页 10 条） | |
| s | 偏移量：默认 -1；翻页用 0、10、20、30… | 实测可取回全量（样本群 36 条全部拿到） |

bkn 算法（社区通用的 CSRF token 计算）：

```python
def bkn(skey: str) -> int:
    h = 5381
    for ch in skey:
        h += (h << 5) + ord(ch)
    return h & 0x7FFFFFFF
```

请求头（我们的实测值）：`User-Agent: Mozilla/5.0 QQ/9.9`、`Referer: https://web.qun.qq.com/`。

## 3. 返回结构

顶层是 JSON。除 `feeds` / `inst` 外，还有 `ui`（发言者 QQ → 昵称/头像 URL 的映射）、`svrt` / `tst` / `group` 等元字段。

两套数组的条目字段名不一样。

`feeds[]`（普通公告）：

```json
{
  "notice_id": "……",
  "sender_id": 10001,
  "publish_time": 1750000000,
  "message": {"text": "……", "image": [], "images": []},
  "settings": {"is_show_edit_card": 0, "remind_ts": 0, "tip_window_type": 0, "confirm_required": 0},
  "read_num": 61
}
```

`inst[]`（「发给新成员」类公告）——字段为缩写：

```json
{
  "u": 10001,
  "fid": "……c70d4868e7490c00",
  "pubt": 1749552583,
  "msg": {"text": "……"},
  "settings": {"is_show_edit_card": 0, "remind_ts": 0, "inst_no_pinned": 0, "tip_window_type": 1, "confirm_required": 0}
}
```

注意：`inst` 条目的 `settings` 里多出一个、且只有它才有的字段——`inst_no_pinned`。

另：每个群最多只能设置一条「发给新成员」公告（即 `inst` 数组至多 1 条）。

## 4. 关键证据（样本）

为保护隐私，群号匿名为 群A / 群B / 群C。

| # | 样本 | 位置 | 发布时间 | 客户端显示 | 接口里的置顶痕迹 |
| --- | --- | --- | --- | --- | --- |
| 1 | 群A「群规」 | inst[0] | 2025-06-10 18:49（fid …c70d4868e7490c00） | 置顶 + 发给新成员（70 人已读） | settings.inst_no_pinned = 0 |
| 2 | 群B「群规/须知」 | inst[0] | 2026-10-03 09:57（fid …f260c06a47aa0800） | 只有「发给新成员」，未置顶 | settings.inst_no_pinned = 1 |
| 3 | 群A「功能性群聊」 | feeds 中一条历史公告 | 2025-06-08 16:18（fid …72474568bee90c00） | 置顶（客户端里就是那条置顶） | 无任何字段可区分 |

- 样本 3 是反面对照：它确实是客户端里的置顶公告之一，但 `feeds` 条目的 `settings` 只有 `is_show_edit_card / remind_ts / tip_window_type / confirm_required` 四个键，读接口没有给出它的置顶状态。
- 反例排除：`tip_window_type = 1` 在多条未置顶的 `feeds` 条目上出现，它不是置顶标记。
- 样本 2 说明：`inst` 不是「置顶列表」，而是「发给新成员」列表；是否置顶由 `inst_no_pinned` 单独表示。
- 样本 1/2 合起来给出映射：`inst_no_pinned = 0` → 置顶中；`= 1` → 未置顶（两群客户端显示与字段值一一对应）。

另：群C 实测 `feeds = 10 / inst = 0`，说明不是每个群都有 `inst` 条目，「发给新成员」是可选功能。

## 5. 写入侧旁证

据我们查阅 NapCat 源码，其设置公告的实现会 POST 到 `https://web.qun.qq.com/cgi-bin/announce/add_qun_notice`，且带一个 `pinned` 参数——说明服务端本身就记录「置顶」这个状态；网页读接口只是没有把它完整吐出来（普通公告没有，`inst` 条目才有）。

## 6. 给 NapCat / OneBot 生态的建议

1. `_get_group_notice` 增加 `inst` 的透传（或合并进返回），让机器人能看到群规类公告；
2. 若可行，把 `inst_no_pinned` 映射成一个 `pinned` 布尔字段（至少对 `inst` 条目）；
3. 支持 `s` 翻页参数（现状：只能拿第一页 10 条）。

## 7. 复现方法（只读）

对**自己账号有权限的群**、以只读方式：

```python
import requests

URL = "https://web.qun.qq.com/cgi-bin/announce/get_t_list"

def bkn(skey: str) -> int:
    h = 5381
    for ch in skey:
        h += (h << 5) + ord(ch)
    return h & 0x7FFFFFFF

def fetch(qid: int, cookies: str, skey: str, s: int = -1, n: int = 10):
    params = {"bkn": bkn(skey), "qid": qid, "ft": 23, "ni": 1,
              "n": n, "i": 1, "log_read": 1, "platform": 1, "s": s}
    headers = {"Cookie": cookies, "User-Agent": "Mozilla/5.0 QQ/9.9",
               "Referer": "https://web.qun.qq.com/"}
    r = requests.get(URL, params=params, headers=headers, timeout=20)
    r.raise_for_status()
    return r.json()  # data["feeds"] / data["inst"]

# 翻页：s=0, 10, 20, 30…（样本群共 36 条历史公告，全部可取回）
```

## 8. 声明

- 以上数据来自对公开网页接口的只读观察（自己的账号、自己的群），未绕过任何鉴权，未修改任何数据。
- 本文不含任何账号凭证、Cookie、聊天内容或个人身份信息；样本群以别名表示。
- 请遵守 QQ 相关协议与当地法律，风险自负。