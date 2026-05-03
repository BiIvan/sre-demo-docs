#1. Запускаем ВМ и ждём её готовность. 
    По готовности получаем такие данные:
    Имя ресурса: r-1node-k8s-module-9-final-879393733
    Данные для подключения к виртуальной машине. IP-адрес -- ааа.ббб.ввв.ггг
    Приватный ключ:
    -----BEGIN OPENSSH PRIVATE KEY-----
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххx
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххх
    -----END OPENSSH PRIVATE KEY-----
    mkdir .ssh
    PowerShell:
    @"
    -----BEGIN OPENSSH PRIVATE KEY-----
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххx
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххх
    -----END OPENSSH PRIVATE KEY-----
    "@ | set-content .ssh\user_key
    
    Bash:
    cat << EOFOE > .ssh/user_key
    -----BEGIN OPENSSH PRIVATE KEY-----
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххx
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххххх
    хххххххх
    -----END OPENSSH PRIVATE KEY-----
    EOFOE
    далее буду писать код для PowerShell, так как в bash для админов всё проще
    

#2. Забираем конфигурацилонный файл k8s с master-ноды
    $IPaddress_ext="ааа.ббб.ввв.ггг"
    mkdir .kube
    scp -i .ssh\user_key ubuntu@$IPaddress_ext:~/.kube/config .\.kube\config
    
    готовим файл конфигурации кластера, согласно инструкции
    (Get-Content .kube\config) -replace '.*certificate-authority-data.*', '    insecure-skip-tls-verify: true' | Set-Content .kube\config.finish -Encoding utf8
    (Get-Content .kube\config) -replace '.*server: https:.*', "    server: https://$IPaddress-ext:6443" | Set-Content .kube\config -Encoding utf8

    прописываем в окружении временно и навсегда
    $env:KUBECONFIG = "$HOME\.kube\config"
    [Environment]::SetEnvironmentVariable("KUBECONFIG", "$HOME\.kube\config", "User")
    
    проверяем доступность кластера
    kubectl cluster-info
    kubectl get nodes
    helm version 
    kubectl get ns
    
    
#3. Раскатываем и настраиваем стек
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    helm upgrade --install mon prometheus-community/kube-prometheus-stack `
      --namespace monitoring --create-namespace `
      --set-string grafana.grafana.ini.feature_toggles.enable=externalServiceAccounts `
      --set alertmanager.alertmanagerSpec.alertmanagerConfigMatcherStrategy.type=None 
 
    получаем пароль Grafana
    $GrafanaPass = kubectl --namespace monitoring get secrets mon-grafana -o jsonpath="{.data.admin-password}"
    $GrafanaPass = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($GrafanaPass))
    echo "Grafana password = $GrafanaPass"
    

#4. Приступаем к сервису OnCall
    создаём канал и получаем его идентификатор (8888888888:ХХХХ-ХХХХХХХХХХХХХХХХХХХХХХХХХХХХХХ), 
    а потом создаём чат и получаем идентификатор чата в этом канале (-1008888888888)
    
    kubectl create namespace oncall
    
    kubectl create secret generic telegram-token --namespace oncall --from-literal=token='8888888888:ХХХХ-ХХХХХХХХХХХХХХХХХХХХХХХХХХХХХХ'
    
    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

    helm upgrade --install oncall grafana/oncall `
      --namespace oncall `
      --set base_url=oncall-engine.oncall.svc.cluster.local:8080 `
      --set base_url_protocol=http `
      --set grafana.enabled=false `
      --set externalGrafana.url=http://mon-grafana.monitoring.svc.cluster.local `
      --set prometheus.enabled=false `
      --set ingress.enabled=false `
      --set ingress-nginx.enabled=false `
      --set cert-manager.enabled=false `
      --set rabbitmq.image.registry=docker.io `
      --set rabbitmq.image.repository=bitnamilegacy/rabbitmq `
      --set redis.image.registry=docker.io `
      --set redis.image.repository=bitnamilegacy/redis `
      --set mariadb.image.registry=docker.io `
      --set mariadb.image.repository=bitnamilegacy/mariadb `
      --set telegramPolling.enabled=true `
      --set oncall.telegram.enabled=true `
      --set oncall.telegram.webhookUrl="" `
      --set oncall.telegram.existingSecret=telegram-token `
      --set oncall.telegram.tokenKey=token `
      --set-string env.DJANGO_DB_CONN_MAX_AGE=\`"0\`" `
      --set-string env.DJANGO_DB_CONN_HEALTH_CHECKS=\`"true\`"
      
    kubectl -n monitoring get deploy,service,pod
    
    $IPaddress_int=kubectl -n monitoring get service | grep service/mon-grafana | Out-String -Stream | ForEach-Object{($_ -split '\s+')[2]}
    
    готовим строку для инсталляции апгрейда OnCall-плагина и применяем её
    $str="'http://admin:$GrafanaPass@$IPaddress_int/api/plugins/grafana-oncall-app/settings' \
      -H \`"Content-Type: application/json\`" \
      -d '{ 
      \`"enabled\`": true, 
      \`"jsonData\`": { 
        \`"stackId\`": 5, 
        \`"orgId\`": 100, 
        \`"onCallApiUrl\`": \`"http://oncall-engine.oncall.svc.cluster.local:8080/\`", 
        \`"grafanaUrl\`": \`"http://mon-grafana.monitoring.svc.cluster.local/\`" 
      } 
    }'
    "
    kubectl -n monitoring exec -it pod/mon-grafana-68c884cd59-kkjm5 -- curl -X -g POST "$str"
    
#5. Развернём пакет Podinfo

    helm repo add podinfo https://stefanprodan.github.io/podinfo
    helm repo update
    helm upgrade --install podinfo podinfo/podinfo `
      --namespace demo --create-namespace `
      --set serviceMonitor.enabled=true `
      --set serviceMonitor.additionalLabels.release=mon
  
    kubectl --namespace demo get pods,services

    @"
    apiVersion: monitoring.coreos.com/v1
    kind: PrometheusRule
    metadata:
      name: podinfo-http-errors
      namespace: demo
      labels:
        release: mon
    spec:
      groups:
        - name: podinfo.rules
          rules:
            - alert: PodInfoHttpErrors
              expr: |
                sum(
                  increase(http_requests_total{namespace="demo", container="podinfo", status=~"4..|5.."}[5m])
                ) > 0
              for: 1m
              labels:
                service: podinfo
                severity: warning
              annotations:
                summary: "PodInfo: 4xx or 5xx HTTP responses detected"
                description: "Podinfo returned non-2xx HTTP responses in the last 5 minutes. Check the pod/service status and application logs."
    "@ | set-content prometheusrule-podinfo.yaml
    kubectl apply -f prometheusrule-podinfo.yaml 
    
            
#6. Развернём документацию, которая тушит пожар
    @"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mkdocs
      namespace: docs
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mkdocs
      template:
        metadata:
          labels:
            app: mkdocs
        spec:
          volumes:
            - name: work
              emptyDir: {}
            - name: site
              emptyDir: {}

          initContainers:
            - name: git-clone
              image: alpine/git:latest
              env:
                - name: REPO_URL
                  value: \`"https://github.com/Yandex-Practicum/sre-demo-docs.git\`"
                - name: REPO_REF
                  value: "main"
              command: ["sh", "-c"]
              args:
                - |
                  set -e
                  rm -rf /work/repo
                  git clone --depth 1 --branch \`"\`$REPO_REF\`" "\`$REPO_URL\`" /work/repo
              volumeMounts:
                - name: work
                  mountPath: /work

            - name: mkdocs-build
              image: squidfunk/mkdocs-material:9
              command: ["sh", "-c"]
              args:
                - |
                  set -e
                  cd /work/repo
                  mkdocs build --site-dir /site
              volumeMounts:
                - name: work
                  mountPath: /work
                - name: site
                  mountPath: /site

          containers:
            - name: nginx
              image: nginx:alpine
              ports:
                - containerPort: 80
              volumeMounts:
                - name: site
                  mountPath: /usr/share/nginx/html
    "@ | set-content deployment.yaml
    
    @"
    apiVersion: v1
    kind: Service
    metadata:
      name: mkdocs
      namespace: docs
    spec:
      selector:
        app: mkdocs
      ports:
        - name: http
          port: 80
          targetPort: 80
    "@ | set-content service.yaml
    
    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    
    kubectl --namespace docs get pods,services
    
    
    
    
    

    
Имя ресурса: r-1node-k8s-module-9-final-879393733
Данные для подключения к виртуальной машине. IP-адрес -- 111.88.146.213.
Приватный ключ:
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZWQyNTUx
OQAAACCvdl3m7EmiCUxO1Y+Gux3TXznGH7FuntIM9U38fntmBAAAAIjtsAqh7bAKoQAAAAtzc2gt
ZWQyNTUxOQAAACCvdl3m7EmiCUxO1Y+Gux3TXznGH7FuntIM9U38fntmBAAAAEBZcCZJrd20k/OD
gleICklC75/CbKfvn2ilmEuH+LoZEK92XebsSaIJTE7Vj4a7HdNfOcYfsW6e0gz1Tfx+e2YEAAAA
AAECAwQF
-----END OPENSSH PRIVATE KEY-----
