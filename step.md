1. kind create cluster
   
   ```bash
   kind create cluster --config 1_kind/kind-cluster.yaml
   ```
3. 設定 K8S 標籤

   1. label infra and taint

      ```bash
      kubectl label nodes exam-worker nodegroup=infra role=infra

      kubectl taint nodes exam-worker role=infra:NoSchedule
      ```

   2. label appliaction and taint

      ```bash
      kubectl label nodes exam-worker2 role=application nodegroup=application
      kubectl label nodes exam-worker3 role=application nodegroup=application

      kubectl taint nodes exam-worker2 role=application:NoSchedule
      kubectl taint nodes exam-worker3 role=application:NoSchedule
      ```

   3. 驗證
      ```bash
      [root@exam-kind-01 ~]# k get no -L nodegroup,role
      NAME                 STATUS   ROLES           AGE   VERSION   NODEGROUP     ROLE
      exam-control-plane   Ready    control-plane   31m   v1.33.1
      exam-worker          Ready    <none>          30m   v1.33.1   infra         infra
      exam-worker2         Ready    <none>          30m   v1.33.1   application   application
      exam-worker3         Ready    <none>          30m   v1.33.1   application   application
      ```
   4. 安裝 helm 為了後續安裝 prometheus-stack

      ```bash
      curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

      helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
      ```

4. 監控安裝

   ```bash
   kubectl create ns monitoring

   helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
     -n monitoring \
     -f 3_setting/prometheus-stack-values.yaml
   ```

5. 安裝外部 grafana

   1. 設定 NodePort 給外部 grafana access
      ```bash
      kubectl apply -f 4_grfana/prometheus-nodeport.yaml
      ```
   2. 安裝 docker compose

      ```bash
      curl -SL https://github.com/docker/compose/releases/download/v2.39.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose

      chmod +x /usr/local/bin/docker-compose
      ```

   3. 持久化 grafana 設定
      ```bash
        mkdir /data/grafana/provisioning
        cp 4_grafana/datasource.yml /data/grafana/provisioning/datasources
        mkdir /data/grafana/data
      ```
   4. 安裝 grafana

      ```bash
        docker-compose -f 4_grafana/docker-compose.yaml up -d

        # 因為permission需要472 權限不夠grafana生長資料夾
        sudo chown -R 472:0 /data/grafana
      ```

6. 設定 Autoscale
   ```bash
   kubectl apply -f 5_statefulset/deployment.yaml
   ```
