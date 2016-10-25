# Docker容器技术学习总结 #

## 12.Rancher##

### 12.1 Rancher 使用问题总结 ###
**问题1：** Rancher Registry仓库设置问题

- **Add Registry:**IP or hostname
- **config INSECURE REGISTRIES:**`/etc/config/docker  --insecure-registry=${DOMAIN}:${PORT}`
- **USING REGISTRIES:** `[registry-name]/[namespace]/[imagename]:[version]`,默认Docker Hub

现有问题：在Rancher上部署