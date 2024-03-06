Software / System Requirements that I used

- minikube v1.32.0
- Kubernetes v1.28.3
- Docker 24.0.7
- Darwin 14.2.1 (arm64)
- Docker container (CPUs=4, Memory=8192MB)

1. Add `127.0.0.1 app.gatsby.local` to your machine’s hosts file
    1. It will map your `localhost` IP address to both hostnames and makes them accessible when  `minikube tunnel` command is ran later on.
        
        ```
        ➜  helm git:(main) ✗ cat /etc/hosts
        ##
        # Host Database
        #
        # localhost is used to configure the loopback interface
        # when the system is booting.  Do not change this entry.
        ##
        127.0.0.1	localhost
        255.255.255.255	broadcasthost
        ::1             localhost
        # Added by Docker Desktop
        # To allow the same kube context to work on the host and the container:
        127.0.0.1 kubernetes.docker.internal
        # End of section
        127.0.0.1 app.gatsby.local # HERE IS THE CHANGE WE WANT
        ```
        
2. Start Minikube + Enable Ingress Addon
    
    ```bash
    minikube start --memory=8192 --cpus=4
    
    ```
    
3. Install metrics-server

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server \
--set addonResizer.enabled=true --set args="{--kubelet-insecure-tls}"
```

1. Install Ingress Controller
    1. Create `ingress-nginx.yaml`
        
        ```yaml
        controller:
          metrics:
            enabled: true
          podAnnotations:
            prometheus.io/scrape: true
            prometheus.io/port: 10254
          autoscaling:
            enabled: true
        ```
        
    2. Run commands
        
        ```yaml
        helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace --values ingress-nginx.yaml
        ```
        
2. Create Secrets
    1. Source `.env`. It is provided with sensitive variables. In production, this would be in a Secrets Manager.
        
        ```
        set -a # automatically export all variables
        source .env
        set +a # stop automatically exporting
        ```
        
        (or just paste this into the terminal)
        
        ```bash
        HOST=app.gatsby.local # if this was a real DNS, i would insert the IP address, not the domain name
        KEY_FILE=private.key
        CERT_FILE=certificate.crt
        
        POSTGRES_USER=postgres
        POSTGRES_PASSWORD=postgres
        ```
        
    2. Create TLS Secrets `secret/certificate`
        
        ```bash
        openssl req -x509 -out ${CERT_FILE} -keyout ${KEY_FILE} \
        -newkey rsa:2048 -nodes -sha256 \
        -subj '/CN=${HOST}' -extensions EXT -config <( \
        printf "[dn]\nCN=${HOST}\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:${HOST}\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
        
        kubectl create secret tls certificate --key ${KEY_FILE} --cert ${CERT_FILE}
        ```
        
    3. Create Postgres Credential Secrets `secret/postgres-secret`
        
        ```bash
        kubectl create secret generic postgres-secret \
        --from-literal=postgresql-username=${POSTGRES_USER} \
        --from-literal=postgresql-password=${POSTGRES_PASSWORD}
        ```
        
3. Install Prometheus for Postgres Monitoring
    1. Create `postgres-exporter.yaml`
        
        ```yaml
        config:
          datasource:
            host: postgres
            userSecret:
              name: postgres-secret
              key: postgresql-username
            passwordSecret:
              name: postgres-secret
              key: postgresql-password
            database: consensys
            sslmode: disable
          # disableDefaultMetrics: true
          annotations:
            prometheus.io/scrape: true
            # prometheus.io/port: 80
        # serviceMonitor:
        #   enabled: true
        
        ```
        
    2. Create `prometheus.yaml`
        
        ```yaml
        serverFiles:
          prometheus.yml:
            scrape_configs:
              - job_name: postgres-prometheus
                scrape_interval: 5s
                # scrape_timeout:  30s
                metrics_path: /metrics
                static_configs:
                  - targets:
                    - postgres-exporter-prometheus-postgres-exporter:80
        alertmanager:
          enabled: false
        prometheus-node-exporter:
          enabled: false
        prometheus-pushgateway:
          enabled: false
        
        ```
        
    3. Create `prometheus-adapter.yaml`
        
        ```yaml
        rules:
          external:
          - seriesQuery: 'pg_stat_activity_count{state="active"}'
            resources:
              template: <<.Resource>>
            name:
              matches: pg_stat_activity_count
              as: pg_stat_activity_count
            metricsQuery: sum(pg_stat_activity_count{state="active"}) by (datname)
        
        prometheus:
          url: http://prometheus-server.default.svc.cluster.local
          port: 80
          path: ""
          
        ```
        
    4. Run commands
        
        ```bash
        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm repo update
        
        # Custom Resource Definitions (CRDs) for ServiceMonitor 
        # LATEST=$(curl -s https://api.github.com/repos/prometheus-operator/prometheus-operator/releases/latest | jq -cr .tag_name)
        # curl -sL "https://github.com/prometheus-operator/prometheus-operator/releases/download/${LATEST}/bundle.yaml" | kubectl create -f -
        
        helm upgrade --install postgres-exporter **prometheus-community/prometheus-postgres-exporter** --values postgres-exporter.yaml
        helm upgrade --install prometheus **prometheus-community/prometheus** --values prometheus.yaml
        helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter --values prometheus-adapter.yaml
        ```
        
4. Install Istio as Service Mesh for secure comm between BE and DB
    
    ```bash
    helm repo add istio https://istio-release.storage.googleapis.com/charts
    helm repo update
    
    helm upgrade --install istio-base istio/base -n istio-system --set defaultRevision=default --namespace istio-system --create-namespace
    helm upgrade --install istiod istio/istiod -n istio-system --wait
    ```
    
5. Run Helm Chart for the Web App
    
    ```bash
    helm upgrade --install webapp ./webapp
    ```
    
6. Start Minikube Tunnel
    
    ```bash
    minikube tunnel
    ```
    
7. Go to [`https://app.gatsby.local/`](https://app.gatsby.local/)