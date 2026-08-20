---
title: 用Python给Steam吃灰游戏库写个随机轮换救赎启动器
published: 2026-08-20
description: 治好大促买完就供着的电子囤积症，让库存里躺平的打折神作重见天日。
image: https://image.astrdark.cyou/file/1772814063613_1770298637_74a94ffb.jpg
tags: [Python, Steam, 游戏日常]
category: 奇思妙想
draft: false
---

每次Steam打折把愿望单清空之后，最神奇的事情就会发生：看着库里暴增的三位数库存发呆十分钟，顺手点开已经玩了八百小时的联机老游戏。花钱买游戏是消费行为，下载安装是体力劳动，点开游玩才是真正的心理博弈。

仓库里那些顶着“好评如潮”光环却游玩时间显示零分钟的作品，正在数字角落里慢慢长毛。既然选择困难症的根源在于给了大脑太多权衡利弊的借口，那就直接把启动权移交给冷酷的算法。

## 解决发呆的加权救赎算法

单纯的完全随机并没有太大意义，遇到体量过大的长篇RPG很容易让人瞬间产生逃避情绪。合理的救赎策略需要考虑三个核心参数：总游玩时长、购买距离天数、以及近期被连续跳过的嫌弃次数。

长时间游玩记录为零的游戏应当获得最高的基准权重，随时间推移逐渐攀升。被随机抽中却被用户手动跳过的倒霉蛋，权重会被强行压制一段时间，避免脚本变成喋喋不休的推销员。

```python
import random
import time
from dataclasses import dataclass

@dataclass
class BacklogGame:
    app_id: int
    name: str
    playtime_minutes: int
    days_since_purchase: int
    skip_penalty: int = 0

    @property
    def redemption_weight(self) -> float:
        if self.playtime_minutes > 120:
            return 0.05
        base = max(1.0, self.days_since_purchase * 0.1)
        freshness_boost = 2.0 if self.playtime_minutes == 0 else 1.0
        penalty_decay = 1.0 / (1.0 + self.skip_penalty * 0.8)
        return (base * freshness_boost) * penalty_decay

class LibraryRedeemer:
    def __init__(self, games: list[BacklogGame]):
        self.games = games

    def pick_next_adventure(self) -> BacklogGame:
        eligible = [g for g in self.games if g.redemption_weight > 0]
        if not eligible:
            raise RuntimeError("所有库存都被你成功逃避了！")
        weights = [g.redemption_weight for g in eligible]
        chosen = random.choices(eligible, weights=weights, k=1)[0]
        return chosen

    def register_skip(self, game: BacklogGame):
        game.skip_penalty += 1
        print(f"【嫌弃记录】已将《{game.name}》打入冷宫，下次权重惩罚已生效。")
```

## 给自己来点微小的物理启动约束

算法挑出了游戏，下一步就是绕过犹豫期。Steam本身支持直接通过URI协议拉起游戏客户端。只要在终端敲下回车确认，脚本就会瞬间调用系统接口唤醒游戏，不给后悔关闭留出多余的操作窗口。

```python
import os
import subprocess
import sys

def launch_steam_game(app_id: int):
    steam_uri = f"steam://run/{app_id}"
    if sys.platform == "win32":
        os.startfile(steam_uri)
    elif sys.platform == "darwin":
        subprocess.run(["open", steam_uri])
    else:
        subprocess.run(["xdg-open", steam_uri])
    print(f"指令已发送，准备进入新世界。")
```

如果在抽中之后五分钟内检测到进程并未启动，脚本会友好地把常玩老游戏的快捷方式临时藏进一个加了时间锁的深层文件夹。虽然防君子不防强行还原，但多出来的这几道繁琐步骤足以打破惯性依赖。

囤积数字资产本质上是在购买某种可能性的入场券，真正体验过后哪怕只玩半小时发现不合胃口，也比永远停留在未拆封的迷茫里踏实得多。
