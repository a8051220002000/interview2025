1. kind create cluster

   ```bash
   kind create cluster --config 1_kind/kind-cluster.yaml
   ```

2. 安裝 helm 為了後續安裝 prometheus-stack

   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   # prometheus 監控使用
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   #  for metics-server
   helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
   ```

3. 監控安裝

   ```bash
   kubectl create ns monitoring
   # 監控
   helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
     -n monitoring \
     -f 3_setting/prometheus-stack-values.yaml
   # 安裝metrics-server hpa 需要用到
   helm upgrade --install metrics-server metrics-server/metrics-server \
      -n monitoring \
      -f 3_setting/metics-server-values.yaml
   # 驗證
   kubectl get po -n monitoring
   ```

4. 設定 K8S 標籤

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
        # 複製dashboard設定至容器內
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
   1. 部屬
   ```bash
   kubectl apply -f 5_statefulset/deployment.yaml
   ```
   2. 確認 auto scale 正常
   ```bash
      [root@exam-kind-01 5_statefulset]# kubectl get hpa -n application
      NAME           REFERENCE              TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
      demo-app-hpa   StatefulSet/demo-app   cpu: <unknown>/50%   2         10        2          16h
      [root@exam-kind-01 5_statefulset]# 
   ```
   3. 驗證 hpa (壓力測試)
   ```bash
      kubectl apply -f 5_statefulset/benchmark.yaml

      # 查看到正常hpa
      [root@exam-kind-01 sysadmin]# k get hpa -n application 
      NAME           REFERENCE              TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
      demo-app-hpa   StatefulSet/demo-app   cpu: 53%/50%   2         10        6          67m
   ```
