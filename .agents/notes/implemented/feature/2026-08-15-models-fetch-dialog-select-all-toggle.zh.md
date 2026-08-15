# Agent Note: Models 获取弹窗中的全选/清空切换

Status: implemented

[English](2026-08-15-models-fetch-dialog-select-all-toggle.md) | 中文

## 问题

`ModelListEditor` 把获取结果敞开成一个平铺的选择框：每个新发现的候选默认勾选，于是对一次报出很多模型的提供方，用户只能一行一行取消勾选；而已配置过的候选以未勾选的行出现，与新候选无从区分。[那条声明记录](../architecture/2026-08-04-declaring-a-provider-from-the-models-page.md)点出了后一半——「已配置过的候选默认不勾选」——它让采纳一次选择变得安全，但同一个呈现仍然留下一个重新添加的入口，选择框仍能经由它覆盖用户已更正的容量。选择框需要针对新发现分组的一个批量控制，而已配置分组需要一种不再充当重新添加目标的呈现。

## 决策

`ModelListEditor` 把发现到的候选拆成两组，并加一个批量切换，全部落在既有的 client 插件与设置面里；询问机制本身不变（见[询问草稿端点](../architecture/2026-08-04-draft-provider-endpoint-interrogation.md)）。

**两组各用自己的行。** `known` 是 profile 的 `models` 里已有的 id 集合。`existing` 收纳落在 `known` 里的 id，逐条渲染为 `ExistingRow`——一个禁用且勾选的复选框加上 id——置于 `fetchGroupExisting` 标题之下；`fresh` 收纳其余 id，逐条渲染为 `CandidateRow`，一个绑定到 `picked` 的活动复选框，置于 `fetchGroupNew` 标题之下。`ExistingRow` 没有切换处理函数、也从不进入 `picked`，因此它是纯展示：说明什么已经配置好，且无法被重新采纳。

**一个切换只作用于新发现分组。** `allPicked` 读 `fresh.every(candidate => picked.has(candidate.id))`，`fresh` 为空时为假，`toggleAll` 在 `fresh` 为空时直接返回。否则它在 `allPicked` 为真时写入 `picked` 减去 `fresh` 的 id，为假时写入 `picked` 加上 `fresh` 的 id。列表上方的按钮（`candidateToolbar`，右对齐）在任一 `fresh` 行未勾选时显示 `fetchSelectAll`，全部勾选后显示 `fetchClearAll`。

**原有的「重新添加护栏」被加强而非替换。** 此前的决定让已配置候选保持为活动的未勾选行，以免采纳覆盖已更正的容量；本记录则让它们彻底不可被重新采纳。这只取代了[在 Models 页声明提供方](../architecture/2026-08-04-declaring-a-provider-from-the-models-page.md)中的呈现方式，而不是那条保证。

## 曾考虑的替代方案

**覆盖整个候选列表的切换。** 这是第一版。它让已配置候选保持为活动的未勾选行，于是「全选」会重新勾选已经配置过的模型，把批量动作变成一条重新添加路径。

**保留已配置候选为活动的未勾选行，只加切换。** 保全了先前的 DOM 形态，但混杂的列表让人看不出什么已经存在，而且切换仍得把它们滤掉——拆成 `existing`/`fresh` 是同一份工作量，换来一个更清楚的结果。

**「全选」与「清空」两个按钮。** 为一个二态动作铺更多 chrome；一个随状态翻转文案的按钮读起来一样，占用的界面却更少。

**改默认值，让新发现的模型默认不勾选。** 超出范围：切换是纯增量的，保留了获取后自动勾选新模型的默认，因为常见情形仍是「把端点服务的模型大多加进来」。

## 后果

报出长模型列表的提供方不再是一场逐行取消勾选的苦差：单击清空新发现的一半，再点一次恢复全选。已添加分组被钉在新分组之上，重新打开时能看清已配置了什么，而不是和新候选混在一起。采纳选择无法覆盖手工更正的容量，如今是因为已配置行根本不可采纳，而不只是默认不勾选。

代价只落在选择框内部：两个分组标题增加了纵向 chrome，而 `ExistingRow` 的复选框是禁用的而非可交互的，因此用户想改的某个容量要在已配置行自己的卡片上编辑，而不是经由获取弹窗。

## 测试

`packages/client/ui-settings-models/tests/provider-form.client.spec.tsx` 新增两个用例、修改一个：端点询问套件里的「全选/清空循环并采纳」，以及「已添加钉在新发现之上、切换只作用于新发现且清理后禁用的已添加行保持不动」。样式 gate 保持 `candidateToolbar` 与 `candidateGroupHead` 在源码中被引用。