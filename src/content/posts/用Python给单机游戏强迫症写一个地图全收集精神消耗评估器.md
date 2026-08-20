---
title: 用Python给单机游戏强迫症写一个地图全收集精神消耗评估器
published: 2026-08-21
description: 为什么明明说好去拯救世界，却在新手村翻了三个小时垃圾桶？给游戏强迫症玩家的跑图精神损耗建立一个数学模型。
image: https://image.astrdark.cyou/file/1772814063613_1770298637_74a94ffb.jpg
tags: [Python, 游戏, 算法, 摸鱼心得]
category: 游戏解构
draft: false
---

魔王在王座上把二郎腿换了七百次，勇者还在新手村隔壁的山头拔杂草。

很多开放世界游戏里都存在这种奇妙的生理现象。地图右上角亮起一个未探索的问号图标，理智告诉你那大概率只是一箱生锈铁钉或者两块硬木板，手里的手柄摇杆却完全不受控制地往悬崖边缘挪动。明知前方全是设计组用来填充工期的边角料，依然要把战争迷雾擦得像刚抛过光的镜子一样透亮。

这种行为在算法视角下，本质是贪心策略在极度稀疏奖励环境下的死锁。只要大脑给未探索区域预设了哪怕0.01%的隐藏神器概率，强迫症就会把遍历整张图的代价当成零成本。

### 跑图掉SAN值的数学抽象

想要量化这种疲惫感，只需要把玩家状态抽象成一个持续失血的消耗模型。每次翻山越岭去开箱子，都会付出体力、注意力与期望落差构成的复合代价。

```python
import math
from dataclasses import dataclass

@dataclass
class MapPOI:
    name: str
    distance: float
    climbing_height: float
    reward_rarity: float  # 0.0 为破烂, 1.0 为传说装备
    is_required_for_trophy: bool

class ExplorerSanityEngine:
    def __init__(self, initial_san: float = 100.0, ocd_level: float = 1.8):
        self.san = initial_san
        self.ocd_level = ocd_level
        self.junk_count = 0

    def evaluate_poi_drain(self, poi: MapPOI) -> float:
        # 地形折磨系数：垂直爬山消耗远大于平地奔跑
        terrain_penalty = poi.distance * 0.1 + math.exp(poi.climbing_height / 50.0)
        
        # 强迫症加权：奖杯强相关内容哪怕再远也必须去
        obsession_weight = self.ocd_level if poi.is_required_for_trophy else 1.0
        
        # 期望落差惩罚：满怀期待跑过去发现是木棍
        disappointment = max(0.0, 1.0 - poi.reward_rarity) * 15.0
        
        total_drain = (terrain_penalty * 0.5 + disappointment) * obsession_weight
        return round(total_drain, 2)

    def visit(self, poi: MapPOI) -> str:
        cost = self.evaluate_poi_drain(poi)
        self.san = max(0.0, self.san - cost)
        if poi.reward_rarity < 0.1:
            self.junk_count += 1
        
        status = "精神焕发" if self.san > 70 else "开始发呆" if self.san > 30 else "变成行尸走肉"
        return f"探索[{poi.name}] 消耗SAN值: {cost}, 剩余SAN值: {self.san:.1f} ({status})"
```

模型里最致命的变量永远是垂直高度。平原上跑五公里尚且能靠自动寻路混过去，一旦目标点刷在九十度垂直绝壁顶端，中途还要躲避三只啄人羽毛的秃鹰，玩家的精神损耗就会呈现指数级暴击。

### 模拟一次地狱级清图路线

拿某款标准罐头开放世界做个简单回测，看看一个满腔热血的强迫症患者在地图西北角能撑几分钟。

```python
engine = ExplorerSanityEngine(initial_san=100.0, ocd_level=1.5)

pois = [
    MapPOI("路边普通木箱", distance=80, climbing_height=0, reward_rarity=0.05, is_required_for_trophy=False),
    MapPOI("悬崖鸟巢（内有闪光石头）", distance=200, climbing_height=120, reward_rarity=0.02, is_required_for_trophy=True),
    MapPOI("水底沉船遗迹", distance=350, climbing_height=0, reward_rarity=0.3, is_required_for_trophy=True),
    MapPOI("雪山之巅眺望点", distance=500, climbing_height=300, reward_rarity=0.1, is_required_for_trophy=True),
]

for poi in pois:
    print(engine.visit(poi))
```

运行输出的结果毫无悬念。刚爬完第二个悬崖鸟巢，背包里塞进两根粗制羽毛和三个野果，SAN值就已经直接腰斩。等好不容易顶着暴风雪滑步登顶雪山，发现山顶宝箱里躺着一把两级前就被淘汰掉的白板铁剑时，系统显示的状态已经正式变更为行尸走肉。

### 治疗代码强迫症的物理手段

游戏厂商把问号洒满地图是为了拉长在线时长，玩家强迫自己把每个问号变成对勾则是落入了虚无的完备性陷阱。全成就奖杯在跳出来的那一秒确实很悦耳，代价却是把本来有趣的探索变成了流水线上的质检打卡。

写完这个脚本后，我果断去游戏设置菜单里关掉了罗盘上的所有微缩雷达与问号提示。

视野变干净之后，路过风景优美的湖畔反而愿意停下来多看两眼水面反光，不必再时时刻刻神经质地转动视角确认山背后有没有漏掉一朵杂草。放过那个挂在树杈上的破箱子，魔王说不定还能在今天傍晚前等到他的决战对手。
