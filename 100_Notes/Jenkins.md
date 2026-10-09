---
tags:
    - 运维/CICD/Jenkins
---

---
# 核心架构
```groovy
pipeline {
    agent any // 1. 在哪里执行

    environment { // 2. 环境变量
        PROJECT_NAME = "my-app"
    }

    stages { // 3. 所有的阶段步骤
        stage('Checkout') { // 阶段一：拉取代码
            steps {
                echo 'Checking out code...'
            }
        }
        stage('Build') { // 阶段二：编译构建
            steps {
                echo 'Building...'
            }
        }
    }

    post { // 4. 构建后的收尾工作（如发送通知）
        always {
            echo 'I will always run!'
        }
    }
}
```
- **`pipeline`**：声明式流水线的顶层标识，所有内容必须写在它内部。
- **`agent`**：指定流水线或某个特定阶段在哪个 Jenkins 节点（Master、Agent 节点或 Docker 容器）上运行。
    - `agent any`：在任何可用的节点上运行。
    - `agent none`：顶层不指定，每个 `stage` 内部必须单独指定自己的 `agent`。
    - `agent { label 'maven-node' }`：在带有 `maven-node` 标签的节点上运行。
    - `agent { docker { image 'node:16-alpine' } }`：直接在指定的 Docker 容器内运行。
- **`stages`**：包裹所有阶段。内部可以包含一个或多个 `stage`。

- **`stage('阶段名称')`**：定义一个独立的逻辑步骤（如：`Checkout`、`Build`、`Test`、`Deploy`），在 Jenkins UI 上会可视化为一个个方块。
- **`steps`**：包含在 `stage` 中，是真正执行具体命令的地方（如执行 Shell 脚本、调用插件）。
- **`post`**：定义根据流水线（或某个 Stage）的执行结果来触发的后续操作。常见条件有：
    - `always`：无论成功还是失败，总是执行（如清理工作空间）。
    - `success`：仅在当前阶段或整条流水线成功时执行（如发送飞书/钉钉成功通知）。
    - `failure`：仅在失败时执行（如发送报警邮件）。

---