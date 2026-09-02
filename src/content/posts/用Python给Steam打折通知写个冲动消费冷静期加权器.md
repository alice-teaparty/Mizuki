---
title: 用Python给Steam打折通知写个冲动消费冷静期加权器
published: 2026-08-18
description: 每次大促销看到绿油油的-80%就管不住手，干脆用几行脚本给入库冲动加上时间衰减和游玩概率权重。
image: https://img.aliceteaparty.top/file/1788337973181_342610b94c7bb64873f4e2a2d042dfe2.png
tags: [Python, 游戏, 杂谈]
category: 瞎折腾
draft: false
---

Steam愿望单里躺着几百个游戏的时候，每逢换季大促，邮箱和弹窗就会变成某种大型心理防线测试现场。很多时候让人掏钱包的根本不是“我今晚就要玩这个”，纯粹是那个绿油油的折扣标签在视觉上太有说服力。买完之后游戏静静躺在库里吃灰，连下载进度条都没走过，真正做到了“买了就当玩过”。

既然人类的自制力在限时打折面前脆弱得像一层单薄的纸，那就把阻力交给代码来提供。写一个微型脚本，拦截打折通知，给每次购买欲望算一个动态加权值，只有综合得分超过阈值才允许放行购买链接。

```python
import math
import time

class ImpulsePurchaseGuard:
    def __init__(self, base_threshold=75.0):
        self.threshold = base_threshold
        self.cool_down_hours = 48

    def calculate_urgency_score(self, discount_pct, original_price, backlog_count, tag_match_ratio, add_days_ago):
        discount_factor = discount_pct * 0.4
        
        # 愿望单放置时间越长，冲动成分越低
        desire_stability = min(math.log1p(add_days_ago) * 10, 30.0)
        
        # 库里未通关存货越多，惩罚权重越大
        backlog_penalty = backlog_count * 1.5
        
        # 偏好标签契合度直接影响放行率
        genre_bonus = tag_match_ratio * 20.0
        
        # 综合欲望得分
        total_score = discount_factor + desire_stability + genre_bonus - backlog_penalty
        return max(total_score, 0.0)

    def evaluate_deal(self, game_name, discount_pct, original_price, backlog_count, tag_match_ratio, add_days_ago):
        score = self.calculate_urgency_score(discount_pct, original_price, backlog_count, tag_match_ratio, add_days_ago)
        print(f"[{game_name}] 折扣: -{discount_pct}% | 愿望单天数: {add_days_ago}天 | 积压库: {backlog_count}部")
        print(f"评估得分: {score:.1f} / 准入阈值: {self.threshold}")
        
        if score >= self.threshold:
            return "放行购买：这确实是你深思熟虑且极大概率会点开的游戏。"
        
        return f"拦截成功：进入 {self.cool_down_hours} 小时冷静期。先把库里的老古董通关两部再来谈新欢。"
```

核心逻辑其实很简单，把折扣率带来的虚假多巴胺拆开，用愿望单的躺平天数和库内积压作品数量来做对冲。某个游戏如果在愿望单里呆了超过半年，哪怕只打了微弱的八折，加权分依然很稳固；要是昨天刚被某个视频切片种草加进愿望单、今天刚好看到骨折促销，在积压库惩罚项的压制下，分数瞬间就会跌破及格线。

把脚本跑在本地的小服务器上，配合Webhook推送。每次收到促销邮件先自动过一遍算分，如果不达标直接吞掉购买直达链接，只留下一条冷冰冰的冷静期倒计时。

跑了几天测试下来，屏幕前那种“不买就亏了几个亿”的幻觉消退得相当彻底。毕竟给欲望套上一层确定性的数学公式后，看着满屏被扣成负分的冲动消费项目，钱包自然就能平安度过每一次大促风暴。
