---
title: 用Python给假面骑士的形态克制链写个拓扑排序与环检测
published: 2026-08-16
description: 为什么特摄剧后期的形态克制链总会变成剪刀石头布甚至因果死循环？用有向有向图的拓扑排序和环检测拆解假面骑士形态进阶与克制悖论。
image: https://image.astrdark.cyou/file/1772814063613_1770298637_74a94ffb.jpg
tags: [Python, 特摄, 假面骑士, 图论, 算法]
category: 技术杂谈
draft: false
---

看假面骑士或者类似特摄剧的时候，大家经常会发现一个特别有意思的战力演化规律。前期剧情里，皮套与强化道具的进阶往往呈现出清晰的线性升级路径：基础形态被初级干部压制，主角反手掏出速度型形态打敏捷克制；遇到高防重装怪，再切换重击力量型形态实现物理破甲。整套战力克制关系严丝合缝，像极了一棵条理分明的有向无环图（DAG）。

随着剧情推进到中后期，剧本编剧和玩具设计师为了给不同派生道具更多的登场高光，各种特殊克制机制开始疯狂堆叠。A形态克制火系怪人，火系怪人克制冰系的B形态，B形态又反过来克制拥有能量护盾的A形态。到了大后期，甚至会出现最终形态被基础形态的某种奇特物理特性反向破防的桥段。整个克制体系从原本干脆利落的有向无环图，悄无声息地演变成了一张充满闭环的复杂有向图。

如果用图论的视角来审视特摄剧里的战斗克制链，我们会发现判定战力体系是否彻底崩坏的最佳手段，正好是计算机科学里最经典的两个算法：拓扑排序（Topological Sort）与环检测（Cycle Detection）。

```python
from collections import defaultdict, deque
from typing import Dict, List, Set, Tuple

class FormCounterGraph:
    def __init__(self):
        self.adj: Dict[str, List[str]] = defaultdict(list)
        self.in_degree: Dict[str, int] = defaultdict(int)
        self.nodes: Set[str] = set()

    def add_counter_relation(self, dominant_form: str, subdued_form: str):
        self.nodes.add(dominant_form)
        self.nodes.add(subdued_form)
        self.adj[dominant_form].append(subdued_form)
        self.in_degree[subdued_form] += 1
        if dominant_form not in self.in_degree:
            self.in_degree[dominant_form] = 0

    def analyze_power_hierarchy(self) -> Tuple[bool, List[str], List[List[str]]]:
        # 基于 Kahn 算法尝试进行拓扑排序
        queue = deque([node for node in self.nodes if self.in_degree[node] == 0])
        topo_order = []
        temp_in_degree = self.in_degree.copy()

        while queue:
            curr = queue.popleft()
            topo_order.append(curr)
            for neighbor in self.adj[curr]:
                temp_in_degree[neighbor] -= 1
                if temp_in_degree[neighbor] == 0:
                    queue.append(neighbor)

        has_cycle = len(topo_order) != len(self.nodes)
        cycles = self._find_all_elementary_cycles() if has_cycle else []
        return not has_cycle, topo_order, cycles

    def _find_all_elementary_cycles(self) -> List[List[str]]:
        # 深度优先搜索检测并回溯所有克制闭环
        visited = set()
        recursion_stack = []
        cycles = []

        def dfs(node: str):
            visited.add(node)
            recursion_stack.append(node)

            for neighbor in self.adj[node]:
                if neighbor not in visited:
                    dfs(neighbor)
                elif neighbor in recursion_stack:
                    cycle_start = recursion_stack.index(neighbor)
                    cycles.append(recursion_stack[cycle_start:] + [neighbor])

            recursion_stack.pop()

        for node in self.nodes:
            if node not in visited:
                dfs(node)
        return cycles
```

在这套拓扑校验模型里，有向边 `A -> B` 代表形态 A 在机制或数值上单向绝对压制形态 B。在一部战斗逻辑严谨的特摄前半程，入度为 0 的节点就是当前战场里的战力天花板（比如刚出场新手保护期拉满的强化过渡形态）。随着拓扑排序一步步将节点剥离，整个梯队层级分明，观众也能清晰看懂为什么主角需要在这个节点切换特定形态。

当剧情进入中后盘乱战阶段，只要把后期所有形态的战绩数据注入模型，Kahn 算法的入度队列往往在运行两步后就陷入停滞。此时算法检测出来的闭环，直接暴露了编剧在战力设计上的因果悖论：

```python
graph = FormCounterGraph()
# 前期常规阶梯升级
graph.add_counter_relation("疾风王牌极限形态", "狂暴加头干部")
graph.add_counter_relation("狂暴加头干部", "尖牙王牌形态")
graph.add_counter_relation("尖牙王牌形态", "普通掺杂体")

# 后期玩具宣发引入的闭环机制
graph.add_counter_relation("月神扳机形态", "高机动飞行怪")
graph.add_counter_relation("高机动飞行怪", "狂暴极限形态")
graph.add_counter_relation("狂暴极限形态", "月神扳机形态")

is_strict_hierarchy, order, paradox_cycles = graph.analyze_power_hierarchy()
```

一旦闭环成立，战力排名的数学传递性就会瞬间瓦解。在严谨的线性拓扑下，`A 胜 B` 且 `B 胜 C` 必然能推导出 `A 胜 C`；但在特摄剧的闭环图里，这套关系直接坍缩成了石头剪刀布，甚至是“人人都有保护期，谁是当集主角谁就自带权值加成”。

从游戏数值和剧作工程的角度来看，这种闭环未必是坏事。纯粹的单向有向无环图会导致旧形态在获得上位替代后彻底沦为仓库废铁，失去任何战术出场意义。而引入克制环与特定场景权重之后，哪怕是最初始的基础形态，也能在特定拓扑闭环里找到重新出场的理由。

写代码和看特摄很多时候有相通的乐趣。我们既沉迷于干净优雅、没有回环与副作用的理想架构，又不得不面对现实世界中为了兼顾各种复杂需求而交织纠缠的状态网络。只要弄清楚背后的图论脉络，就算战力体系乱成一团乱麻，也能看得津津有味。