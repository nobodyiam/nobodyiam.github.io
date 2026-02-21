---
layout:     post
title:      开源社区的自动驾驶模式：我用 AI 将 Apollo 发版效率提升了 10 倍
date:       2026-02-21 09:00:00 +0800
summary:    与 Codex 结对重构 Apollo 发版流程的 5 天里，我看到了 AI 在探索未知域（Unknown-Unknowns）的惊艳表现。纯靠算力驱动，我们初步趟通了开源社区的自动驾驶模式。软件工程的下一个时代，属于高阶的 Taste 与创造力。
categories:
---
# 缘起

春节期间难得清闲，看着近期火热的 [OpenClaw](https://github.com/openclaw/openclaw) 几乎一天一个版本的发布节奏，我突然意识到——[Apollo](https://github.com/apolloconfig/apollo) 已经整整一年没发新版本了😂。反思下来，懒确实是首要原因，但更深层次的问题是：Apollo 的发版成本实在太高了。

## 版本验证：大量手工操作
+ 后端代码有单测和集测覆盖，但页面功能还得人工逐个点击验证
+ Apollo 支持多种登录方式（Spring Security、LDAP、OIDC），每种都需要手动搭建环境、构造测试数据
+ 多种部署模式（Docker、Helm Chart）也要逐一验证

## 发布流程：繁琐且冗长
+ 提炼版本内容撰写 Release Note
+ 版本打包、构建镜像、更新 Helm Chart
+ PMC 投票
+ 撰写版本发布公告

回顾以往的几次发版，一次完整的发布基本需要投入整整 2 天时间。对于用业余时间参与开源社区的贡献者来说，Review Issue 和 PR 还能利用碎片时间，但版本发布这种需要大块时间投入的工作，自然就成了能拖就拖的事情。

我相信大多数开源社区都面临类似的困境——人力投入有限，如果某个环节（比如发版）成本过高，就很难持续坚持。

解药其实很明确：**让繁琐的工作尽可能自动化**。结合近期 AI 在编程领域的迅猛发展，我越发相信开源社区的日常工作完全能够实现自动化——解答 Issue、Review PR、版本发布，甚至未来实现 Issue to PR，让社区运营真正进入“自动驾驶”模式。于是，我决定以 Apollo 2.5.0 的版本发布为试点，尝试打通版本验证和发版流程的自动化。

# 成果

和 Codex（`gpt-5.3-codex xhigh`）进行了 5 天高密度的 Pair Programming（其实主要是“指挥”），消耗了 10 亿 Token，在我没写一行代码的情况下（甚至连打字都很少，全程语音输入），取得了以下阶段性成果。

## 版本验证自动化
1. **主链路 UI 操作的端到端测试**（创建 App/Cluster/Namespace、配置修改/发布/回滚/灰度等），PR：[https://github.com/apolloconfig/apollo/pull/5551](https://github.com/apolloconfig/apollo/pull/5551)
    1. ![](/images/2026-02-21/e2e-ui-test-pr.png)
2. **Portal 不同登录方式的集成测试**（LDAP、OIDC），PR：[https://github.com/apolloconfig/apollo/pull/5557](https://github.com/apolloconfig/apollo/pull/5557)
    1. ![](/images/2026-02-21/portal-auth-integration-test-pr.png)
3. **Docker 和 Helm Chart 部署的冒烟测试**，PR：[https://github.com/apolloconfig/apollo/pull/5558](https://github.com/apolloconfig/apollo/pull/5558)、[https://github.com/apolloconfig/apollo-helm-chart/pull/24](https://github.com/apolloconfig/apollo-helm-chart/pull/24)
    1. ![](/images/2026-02-21/docker-smoke-test-pr.png)
    2. ![](/images/2026-02-21/helm-chart-smoke-test-pr.png)

有了这些自动化验证，每次 PR 提交时都会自动执行全链路功能测试，项目**时刻保持在可发布状态**，自然也就不需要为发版而单独做版本验证了。

## 发布流程自动化
我借助 Codex 沉淀了 3 个 Release Skill（详见 [Apollo 发布指南](https://www.apolloconfig.com/#/zh/contribution/apollo-release-guide)），通过自然语言触发 Codex/CC 等 Coding Agent 完成发布流程。在关键动作执行前，Agent 会等待确认（如 Release Note 文案、版本更新 PR 等）。

1. **`apollo-java-release` Skill（发布 `apolloconfig/apollo-java`）**
    + **自动完成**：版本 Bump PR → 从 `CHANGES.md` 生成 Release Notes 并创建 Pre-release → 发布 jar 包到 Maven 仓库 → Pre-release 转正式 Release → 发布版本公告 → 更新代码版本到下一个 SNAPSHOT

2. **`apollo-release` Skill（发布 `apolloconfig/apollo`）**
    + **自动完成**：版本 Bump PR → 从 `CHANGES.md` 生成 Release Notes 并创建 Pre-release → 打包代码上传 → 发布 Docker 镜像到 Docker Hub → Pre-release 转正式 Release → 发布版本公告 → 更新代码版本到下一个 SNAPSHOT

3. **`apollo-helm-chart-release` Skill（发布 `apolloconfig/apollo-helm-chart`）**
    + **自动完成**：Chart 版本变更检测与一致性校验 → `helm lint` / `package` / `repo index` → 变更白名单校验 → 生成分支与 Commit 草案

有了这些 Skill，发版真正实现了“无脑操作”：不用为 Release Note 犯愁，不用记忆繁琐的发布步骤。触发 Skill 后，人需要做的就是确认 Agent 的产出物是否 OK（比如 Release Note 内容），过程中可以通过自然语言提出建议和想法，剩下的就是一路点确认。至于 PMC 投票——它原本就是为了人工交叉验证，在自动化能确保版本发布正确性的前提下，这个环节也可以逐步弱化。

# 感悟

通过这套自动化流程，Apollo 后续的版本发布时长将缩短到 1 小时以内，相比以往 2 个整天的投入，**效率提升了至少 10 倍**。

但这还不是最让我兴奋的。最近每天指挥 Codex 干活，让我对 AI Coding 有了全新的认识：

过去，我认为 AI 主要是帮我们做脏活累活，把繁杂工作自动化。比如 Apollo 社区呼声很高的升级 Spring Boot 3.x（是的，Codex 已经做完了，下个版本发，PR: [https://github.com/apolloconfig/apollo/pull/5555](https://github.com/apolloconfig/apollo/pull/5555)）。

![](/images/2026-02-21/spring-boot3-upgrade-pr.png)

但现在，我意识到 AI Coding 真正的价值在于**极大地扩展个人的能力边界**。以这次的版本验证自动化为例：

+ UI 端到端测试和 LDAP、OIDC 自动化集成——我知道有解决方案，但之前没做过，现学现做可能要半个月。AI 帮我省下了这些时间，这还属于 **“我知道自己不知道”的范畴（Known-Unknowns）**。
+ Helm Chart 部署的冒烟测试——我原以为 Codex 会给我写一堆脚本，在我本地执行 K8s 部署，再拉起 MySQL 和 Apollo 验证。结果看到它写出来的 `yml` 文件，我惊呆了：原来 GitHub Action 拉起的容器里可以部署 K8s（kind）？！还能这么玩？更绝的是，它自己找到并使用了 Apollo 的 `sql` 文件（我从没提过这个信息），就这么平静地写出了一套 **“我不知道自己不知道”的方案（Unknown-Unknowns）**。这种惊喜感，正是让我沉浸其中、无法自拔、晚上都睡不好觉的原因。
    - ![](/images/2026-02-21/kind-workflow-yaml.png)

Codex 就像一个极其专业的 Pair Programmer，和它对话很爽，让我重拾了 Coding 的乐趣。虽然偶尔会有点“降智”（可能是 Context 太长？），但总体非常在线，还不定期带来惊喜。这让我感慨：未来的瓶颈可能在人，而不在 Agent。

几点感悟：

1. **纯 Coding 的工作已经步写作的后尘**，成为被 AI 深度掌握的领域——它做得比大多数人都专业，技术门槛大幅下降。
2. **人依然是必需的，但主要提供高阶能力（Taste）**：比如要解决什么问题、判断解法是否合适、与实际业务场景的结合等。未来需要的是创造力和想象力——想象一下每个人带 50 个 Agent 干活的场景。
3. **实操层面的经验**：
    - 复杂工作先开启 Plan Mode，让 AI 做方案设计和计划，再进入编码阶段。
    - 构建好环境，让 AI 无障碍调用环境中的工具链（在 Apollo 的例子中是 `gh` / `git` / `mvn` / `docker` / `npm` / `python` / `java` / `helm` 等）。
    - 构建测试验证集，让 AI 知道自己做得对不对（当然，测试集本身也可以让 AI 来构建）。

# 展望

1. **算力将成为新的稀缺资源**——最近我每天消耗几亿 Token，未来随着应用场景扩大，算力需求还会指数级增长。如果说农业时代的生产资料是土地，工业时代是能源，那么 AI 时代的生产资料就是算力。开源社区的基础设施投入，也可能要从“托管服务器”转向“托管算力”。
2. **项目管理的计量单位从“人天”变成“Token”**——当 AI 成为主要劳动力，我们评估一个任务的工作量，可能不再问“这需要几个人干几天”，而是“这需要消耗多少 Token”。项目管理工具里可能会出现“本月剩余 Token 配额”这样的指标，预算也从“人力成本”转向“算力成本”。
3. **开源社区进入“自动驾驶模式”是大概率事件**——随着更多 Skills 的沉淀（比如 Apollo 正在积累的 `issue-review`、`issue-to-pr`、`pr-review` 等），社区日常运营的绝大多数环节都可以交给 AI 代理。贡献者将更多地扮演“产品经理”和“架构师”的角色：定义问题、评审方案、把握方向，而不是陷在具体执行里。想象一下：一个 PR 提交后，AI 自动 Review、触发测试、提供修改建议；一个 Issue 被提出后，AI 自动分析、生成代码、提交 PR——人只需要在关键节点做确认。这不再是科幻，而是正在发生的现实。

开源社区的“自动驾驶”时代，正在加速到来。
