---
tags:
    - 运维/prometheus
---

> [!note]
> - 不负责计算告警是否触发（那是 Prometheus 的工作）
> - 而是==专门负责接收 Prometheus 发来的**原始告警流**，并对这些告警进行**去重、分组、抑制、静默**等降噪处理，最终将干净、清晰的告警投递到钉钉、企业微信、邮件或 PagerDuty 等通知媒介==

---
# 工作流程

```
 ┌──────────────┐      ┌────────────────┐      ┌────────────────┐
 │  Prometheus  ├─────►│ 1. 接收与去重  ├─────►│   2. 告警分组  │
 └──────────────┘      └────────────────┘      └───────┬────────┘
                                                       │
                                                       ▼
 ┌──────────────┐      ┌────────────────┐      ┌────────────────┐
 │ 5. 投递通知  │◄─────┤ 4. 路由与通知  │◄─────┤ 3. 静默与抑制  │
 └──────────────┘      └────────────────┘      └────────────────┘
```
1. 接收与去重 (Deduplication)
   - **接收**：Prometheus 发现异常后，会周期性（如每 15 秒）向 Alertmanager 的 HTTP 接口发送处于 `Firing`（激活）状态的原始告警。
   - **去重**：在高可用架构中，通常会有两个（或多个）完全相同的 Prometheus 实例监控同一个目标，它们会同时发出相同的告警。Alertmanager 内部通过计算告警标签的哈希值，自动将这些重复的告警**合并为一条**，确保不会收到双份通知。

1. 分组 (Grouping)
   - **化零为整**：如果集群中某个核心交换机挂了，可能会导致几十个微服务同时报错，瞬间产生上百条告警。
   - **机制**：Alertmanager 会根据用户配置的维度（例如 `group_by: ['alertname', 'cluster', 'service']`）将相关的告警打包成一个“组”。
   - **效果**：系统不会连发 100 条短信把你的手机轰炸到死机，而是把它们聚合成**一条包含 100 个故障 Pod 列表的摘要邮件**发给你。
1. 静默与抑制 (Silences & Inhibition)
   - **静默 (Silences)**：**主动临时关闭**。比如运维人员要在凌晨 2 点升级数据库，可以在 Alertmanager Web UI 上创建一个静默规则（匹配 `service="mysql"`），在这段维护时间内，该数据库的所有告警都会被拦截，不会打扰到其他人。
   - **抑制 (Inhibition)**：**依赖级级联关闭**。比如当一个机房（`datacenter="A"`）的网络挂了（触发 `NetworkDown` 告警），那么该机房内所有服务器、应用的无法访问告警（如 `InstanceDown`）都会被自动抑制。因为根源是网络，应用报警只是噪音。
1. 路由 (Routing Tree)
   - Alertmanager 内部维护着一棵**路由树 (Routing Tree)**。
   - 所有的告警组进入路由树后，会从根节点开始，根据告警的标签（Labels）进行匹配。
   - **例如**：
     - 带有 `severity="critical"`（紧急）标签的告警，被路由到**短信/电话**通道，直接叫醒值班运维。
     - 带有 `severity="warning"`（警告）标签的告警，被路由到**企业微信/钉钉**群机器人，白天上班看一眼就行。
     - 属于 `team="dev-a"` 的告警，精准发送给 **A 研发团队**，不打扰 B 团队。
1. 投递通知 (Receivers)
   - 最后，经过降噪、分类的告警信息会被转化为对应接收介质（Webhook, Email, Slack 等）所需的格式，正式发送出去。

---
# 配置
## 传统
```yml
global:
  resolve_timeout: 5m # 如果5分钟内没有收到新告警，自动标记为已恢复
  smtp_smarthost: '://qq.com' # 邮件服务器地址
  smtp_from: 'alert@yourdomain.com'
  smtp_auth_username: 'alert@yourdomain.com'
  smtp_auth_password: 'your-smtp-password'
  smtp_require_tls: false

# 1. 路由树定义（告警传进来后，从上到下匹配）
route:
  group_by: ['alertname', 'cluster', 'service'] # 按照这三个标签进行分组聚合
  group_wait: 30s      # 初次告警等待30秒，看有没有同组告警一起触发
  group_interval: 5m   # 同一组内有新告警加入时，等待5分钟再发新通知
  repeat_interval: 12h # 故障未解决，每12小时重复发送一次
  receiver: 'default-receiver' # 默认接收人（未匹配到任何子路由时使用）

  # 子路由（Routes）
  routes:
  - match:
      severity: critical # 匹配紧急告警
    receiver: 'ops-phone-webhook' # 发送到电话/严重告警通道
    continue: true                # 继续向下匹配，同时发邮件
    
  - match_re:
      service: ^(user-service|order-service)$ # 正则匹配特定核心业务
    receiver: 'dev-team-a'

# 2. 接收人通道具体配置（Receivers）
receivers:
- name: 'default-receiver'
  email_configs:
  - to: 'devops@yourdomain.com'
    send_resolved: true # 告警恢复时是否发送通知

- name: 'ops-phone-webhook'
  webhook_configs:
  - url: 'http://dingtalk-webhook-adapter:8060/dingtalk/ops/send' # 钉钉转发网关
    send_resolved: true

- name: 'dev-team-a'
  webhook_configs:
  - url: 'http://wechat-webhook-adapter:8060/wechat/team-a/send' # 企业微信转发网关
    send_resolved: true

# 3. 抑制规则（Inhibition Rules）
inhibit_rules:
  - source_match:
      alertname: 'NodeNetworkDown' # 当网络挂了（源告警触发）
    target_match:
      alertname: 'InstanceDown'    # 抑制实例无法访问告警（目标告警被静音）
    equal: ['node', 'instance']    # 确保是同一个节点上的故障

```

## prometheus operator (AlertmanagerConfig)
1. 配置业务团队自身的告警路由 (`AlertmanagerConfig`)
   假设这是在 `dev-team-a` 命名空间下配置的告警接收策略：
   ```yml
    apiVersion: ://coreos.com
    kind: AlertmanagerConfig
    metadata:
      name: team-a-alert-config
      namespace: dev-team-a # 仅对该命名空间下的告警生效
      labels:
        release: kube-prometheus-stack # 确保能被 Prometheus 实例的 alertmanagerConfigSelector 匹配
    spec:
      route:
        groupBy: ['alertname', 'service']
        groupWait: 30s
        groupInterval: 5m
        repeatInterval: 6h
        receiver: 'team-a-dingtalk' # 默认发到团队钉钉
        routes:
        - matchers:
          - name: severity
            value: critical
          receiver: 'team-a-email-critical' # 紧急告警额外抄送邮件
      receivers:
      - name: 'team-a-dingtalk'
        webhookConfigs:
        - url: 'http://cluster.local'
          sendResolved: true
      - name: 'team-a-email-critical'
        emailConfigs:
        - to: 'team-a-leader@yourdomain.com'
          sendResolved: true
          # 认证密码等敏感信息建议引用 K8s Secret
          authPassword:
            name: alertmanager-smtp-secret
            key: password
   ```

2. 配套的密码凭证 (`Secret`)
   针对邮件密码等敏感信息，在同命名空间下创建 K8s Secret：
   ```yml
    apiVersion: v1
    kind: Secret
    metadata:
      name: alertmanager-smtp-secret
      namespace: dev-team-a
    type: Opaque
    stringData:
      password: "your-email-token-or-password"
   ```
3. 配套的告警规则 (`PrometheusRule`)
   将上述路由与告警规则结合，业务团队只需在自己的 Namespace 下再部署一个 `PrometheusRule`，带上对应的标签即可触发上述路由：
   ```yml
    apiVersion: ://coreos.com
    kind: PrometheusRule
    metadata:
      name: team-a-business-alerts
      namespace: dev-team-a
      labels:
        release: kube-prometheus-stack # 确保被 Prometheus 的 ruleSelector 匹配
    spec:
      groups:
      - name: app-error-rates
        rules:
        - alert: Http5xxRateTooHigh
          expr: sum(rate(http_requests_total{status=~"5.*"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100 > 5
          for: 2m
          labels:
            severity: critical # 触发上面的邮件额外抄送路由
            service: order-service
          annotations:
            summary: "服务 {{ $labels.service }} 5xx 错误率过高"
            description: "当前 5xx 错误率已超过 5%，持续 2 分钟。"
   ``` ^d81f5e ^74b56f