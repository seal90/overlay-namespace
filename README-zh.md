# 叠加命名空间
## 概念
**叠加命名空间（Overlay Namespace）** 是一种基于标签亲和性的服务发现与资源配置机制。它在现有集群中为部分服务实例添加特定标签，构建出逻辑隔离的“叠加层”；未标记的实例则作为默认后备（default fallback）。该机制统一适用于**服务调用、消息消费、定时任务**等场景，并支持**分层配置加载**。

**服务发现策略**

+ **无标签请求**：仅路由至默认实例，确保基础命名空间隔离；
+ **带标签请求**：优先选择标签匹配的实例；若无可用实例，则安全回退至默认实例。

**配置加载策略**

+ **无标签服务**：仅加载默认命名空间的配置；
+ **带标签服务**：优先加载当前叠加命名空间的配置，缺失项自动回退至默认命名空间，实现 **“覆盖 + 继承”** 的组合语义。

该机制无需物理隔离，即可实现轻量级命名空间划分（如灰度发布、测试环境或区域路由），同时保障系统高可用性与跨组件行为的一致性。

该实现的效果是：在基础服务之上，仅需添加少量带标签的服务实例，即可划分出一个逻辑独立的新命名空间（相当于构建了一个轻量级“集群”）。这一机制与 OverlayFS 非常相似——后者通过在基础镜像上叠加少量文件层来生成新的镜像。因此将其命名为 **叠加命名空间（Overlay Namespace）**

## 流程例子
假设系统中存在一个**默认命名空间**（黑色，即主命名空间），以及三个**叠加命名空间**：`Purple`、`Green` 和 `Red`。命名空间标签由 **Gateway 统一注入与透传**，并在服务调用链中全程携带。

各场景行为如下：

+ **Purple 命名空间**  
上游请求携带命名空间标签 `Purple`，Gateway 将该标签透传至下游服务。由于 `Purple` 命名空间部署了**全量服务**，所有请求均在该命名空间中闭环处理，**不会回退到默认命名空间**。
+ **Green 命名空间**  
上游请求携带标签 `Green`，Gateway 透传该标签。由于 `Green` 命名空间**仅部署部分服务**，请求会按以下规则路由：
    - 若目标服务在 `Green` 中存在 → 路由至 `Green` 实例；
    - 若目标服务在 `Green` 中不存在 → **自动回退至默认命名空间**的实例。
+ **Black（默认）命名空间**  
上游请求**未携带命名空间标签**，且 `user_id`**未命中**`user_id % 100` 的分流规则，请求进入默认命名空间。此类请求**严格限制在默认命名空间内**，**不会访问任何叠加命名空间**的实例。
+ **Red 命名空间（基于规则触发）**  
上游请求**未携带命名空间标签**，但 `user_id`**命中**`user_id % 100` 的分流规则，Gateway **自动注入** `Red` **标签**。后续行为与 `Green` 类似：
    - `Red` 中存在的服务 → 路由至 `Red` 实例；
    - `Red` 中缺失的服务 → 回退至默认命名空间实例。

![overlay_namespace_example](./pic/overlay_namespace_example.jpg)

## 相似机制
| 产品 | 实现 | 使用 |
| --- | --- | --- |
| Kubernetes | Topology | • Service 配置 topologyKeys<br/>• 依据请求当前所在Pod上的标签与目标服务做匹配<br/>• 可通过显式规则实现 fallback 到所有 |
| Istio | DestinationRule+Subset | • 通过 subset 为服务实例打标签<br/>• VirtualService 可配置根据请求内容将其路由到特定 subset<br/>• 可通过显式规则实现 fallback 到特定实例 |
| Consul | Intentions with Service Tags | • 为服务实例添加 tags<br/>• 请求可通过 ?tag=prod 指定只访问带该 tag 的实例<br/>• 不支持自动 fallback —— 如果指定 tag 不存在，请求直接失败 |
| Envoy  | Locality-based Routing | • 支持按 locality（区域、可用区）划分 endpoint 优先级<br/>• 高优先级 locality 无健康实例时，自动 failover 到低优先级 |
| Spring cloud loadbalancer | ApiVersion | • 服务配置 API_VERSION 属性<br/>• 请求中的信息与目标服务属性信息做匹配<br/>• 可配置 fallback 到所有实例 |
|  | HintBased | • 服务配置 hint 属性<br/>• 请求中的信息与服务属性信息做匹配<br/>• fallback 到所有实例 |
|  | RequestBasedStickySession | • 请求中cookie的信息与服务instance id做匹配<br/>• fallback 到所有实例 |
|  | Subset | • 在服务实例中获取一部分实例 |
|  | ZonePreference | • 服务配置 zone 属性<br/>• 本服务的 zone 信息与目标服务 zone 信息匹配<br/>• fallback 到所有实例 |