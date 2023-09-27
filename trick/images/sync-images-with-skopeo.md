# 使用 skopeo 批量同步 helm chart 依赖镜像

## skopeo 是什么？

[skepeo](https://github.com/containers/skopeo) 是一个开源的容器镜像搬运工具，比较通用，各种镜像仓库都支持。

## 安装 skopeo

参考官方的 [安装指引](https://github.com/containers/skopeo/blob/main/install.md)。

## 导出当前 helm 配置依赖哪些镜像

```bash
$ helm template -n monitoring -f kube-prometheus-stack.yaml ./kube-prometheus-stack | grep "image:" | awk -F 'image:' '{print $2}' | awk '{$1=$1;print}' | sed -e 's/^"//' -e 's/"$//' > images.txt
$ cat images.txt
quay.io/prometheus/node-exporter:v1.3.1
quay.io/kiwigrid/k8s-sidecar:1.19.2
quay.io/kiwigrid/k8s-sidecar:1.19.2
grafana/grafana:9.0.2
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.5.0
quay.io/prometheus-operator/prometheus-operator:v0.57.0
quay.io/prometheus/alertmanager:v0.24.0
quay.io/prometheus/prometheus:v2.36.1
bats/bats:v1.4.1
k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
```

* 使用 helm template 渲染 yaml，利用脚本导出所有依赖的容器镜像并记录到 `images.txt`。
* 可以检查下 `images.txt` 中哪些不需要同步，删除掉。

## 准备同步脚本

准备同步脚本(`sync.sh`):

```bash
#! /bin/bash

DST_IMAGE_REPO="registry.imroc.cc/prometheus"

cat images.txt | while read line
do
        while :
        do
                skopeo sync --src=docker --dest=docker $line $DST_IMAGE_REPO
                if [ "$?" == "0" ]; then
                        break
                fi
        done
done
```

* 修改 `DST_IMAGE_REPO` 为你要同步的目标仓库地址与路径，`images.txt` 中的镜像都会被同步到这个仓库路径下面。

赋予脚本执行权限:

```bash
chmod +x sync.sh
```

## 登录仓库

同步镜像时，不管是源和目的，涉及到私有镜像，都需要先登录，不然同步会报错。

登录方法很简单，跟 `docker login` 一样，指定要登录的镜像仓库地址:

```bash
skopeo login registry.imroc.cc
```

然后输入用户名密码即可。

## 执行同步

最后执行 `./sync.sh` 即可将所有镜像一键同步到目标仓库中，中途如果失败会一直重试直到成功。

## FAQ

### 为什么不用 skopeo 配置文件方式批量同步？

因为配置相对复杂和麻烦，不如直接用一个列表文本，每行代表一个镜像，通过脚本读取每一行分别进行同步，这样更简单。