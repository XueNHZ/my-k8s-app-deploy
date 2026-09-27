# my-k8s-app-deploy

GitOps 部署仓库。仓库保存 Helm chart 和 Argo CD Application 模板；真实 GitHub、Harbor 地址和凭据在运行时注入。

## 目录

- application.yaml：Argo CD Application 模板。
- charts/my-app：应用 Helm chart。
- charts/my-app/values.yaml：镜像、资源和服务配置。
- charts/my-app/templates：Deployment 和 Service 模板。
- test-pull.yaml：可选的 Harbor 拉取测试 Pod。

## 模板变量

application.yaml 使用：

    ${DEPLOY_REPO_URL}

values.yaml 和测试清单使用：

    ${HARBOR_REGISTRY}/${HARBOR_PROJECT}/my-app

Argo CD 不会自动读取 shell 环境变量，请先渲染模板。

## 本地渲染

    cd ~/projects/my-k8s-app-deploy
    export DEPLOY_REPO_URL="https://github.com/OWNER/DEPLOY_REPOSITORY.git"
    export HARBOR_REGISTRY="registry.example.internal:80"
    export HARBOR_PROJECT="library"

    envsubst < application.yaml > /tmp/application.rendered.yaml
    kubectl apply -f /tmp/application.rendered.yaml

    envsubst < charts/my-app/values.yaml > /tmp/values.rendered.yaml
    cat /tmp/values.rendered.yaml

/tmp 下的渲染文件不要提交到 Git。

## Harbor imagePullSecret

集群中需要名为 harbor-secret 的 Secret。不要把用户名密码写入 YAML：

    kubectl create namespace my-app
    kubectl -n my-app create secret docker-registry harbor-secret --docker-server="$HARBOR_REGISTRY" --docker-username="$HARBOR_USERNAME" --docker-password="$HARBOR_PASSWORD"

HTTP Harbor 还要求 Kubernetes 节点上的 containerd 或 Docker 配置 insecure registry。

## 提交和同步

    git status
    git diff
    git add application.yaml charts README.md test-pull.yaml
    git add charts/my-app
    git commit -m "describe the deployment change"
    git push origin main

应用仓库 workflow 会自动更新 values.yaml 的 image.tag。Argo CD 根据 automated syncPolicy 同步集群。

提交前检查：

    git grep -n -I -e "172." -e "password" -e "token" HEAD
