## 资源清单格式

```yaml
piVersion: group/apiversion  # 如果没有给定 group 名称，那么默认为 core，可以使用 kubectl api-versions # 获取当前 k8s 版本上所有的 apiVersion 版本信息( 每个版本可能不同 )
kind:       #资源类别
metadata：  #资源元数据  
  name   namespace   
  lables   annotations   # 主要目的是方便用户阅读查找
spec: # 期望的状态（disired state）
status：# 当前状态，本字段由 Kubernetes 自身维护，用户不能去定义
```

