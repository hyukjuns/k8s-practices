# Install Loki on k8s cluster
Helm 을 사용하여 Loki를 클러스터에 설치합니다.

## Step
1. Create namespace

    ```
    k create ns loki
    ```

2. Add Repo

    ```
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update

    # 설치할 버전 확인
    helm search repo grafana/loki --versions
    ```

3. Configure Values

    ```
    helm show values grafana/loki > values.yaml
    
    # vi user-values.yaml
    ...
    loki:
      storage:
        bucketNames:
    ...
    ```

4. Install

    ```
    helm install loki grafana/loki --values ./values/user-values.yaml -n loki --version 5.15.0
    ```

### Ref
- Loki github

    https://github.com/grafana/loki

- Install Guide

    https://grafana.com/docs/loki/latest/installation/helm/install-scalable/