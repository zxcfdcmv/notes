---
tags:
    - 运维/K8S
---
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  namespace: default
  labels:
    app: my-app
    tier: backend
spec:
  # 副本数：期望运行的 Pod 数量
  replicas: 3
  
  # 选择器：Deployment 管理哪些 Pod（必须匹配 Pod 模板中的 labels）
  selector:
    matchLabels:
      app: my-app
      tier: backend
  
  # 更新策略：滚动更新配置
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 滚动更新时最多比期望副本数多 1 个 Pod
      maxUnavailable: 0  # 滚动更新时最多允许 0 个 Pod 不可用（保证服务不中断）
  
  # Pod 模板（核心）
  template:
    metadata:
      labels:
        app: my-app
        tier: backend
    spec:
      # 容器列表（可多个）
      containers:
      - name: my-container
        image: nginx:1.21
        imagePullPolicy: IfNotPresent  # Always | Never | IfNotPresent
        
        # 端口配置
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        
        # 资源限制（生产环境必须设置）
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        
        # 环境变量
        env:
        - name: ENV_NAME
          value: "production"
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: my-secret
              key: password
        
        # 健康检查：存活探针（重启不健康的容器）
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        
        # 健康检查：就绪探针（控制是否接收流量）
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # 挂载存储卷
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        - name: data-volume
          mountPath: /data
      
      # 卷定义（与 containers 同级）
      volumes:
      - name: config-volume
        configMap:
          name: my-config
      - name: data-volume
        persistentVolumeClaim:
          claimName: my-pvc
      
      # 节点调度相关（可选）
      nodeSelector:
        disktype: ssd
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      
      # 优雅终止时间（秒）
      terminationGracePeriodSeconds: 30
```