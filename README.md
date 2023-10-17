# K3S with example application (golang, longhorn, sentry) ingressroute stripPrefix

**Setup k3s :**
> k3s deployed on machine with 3 nodes 1 master and 2 Worker. The data dir change to --data-dir=/data/k3s 
- Master-1 Node **Run on VM Master-1**
  > when create master node clustering need to add parameter --cluster-init on the first master node
  ```bash
  curl -sfL https://get.k3s.io | K3S_NODE_NAME=master-1  sh -s - server --cluster-init --data-dir=/data/k3s
  kubectl get nodes -o wide
  cat /data/k3s/server/node-token
  ```
- Master-2 Node **Run on VM Master-2**
  ```bash
  curl -sfL https://get.k3s.io | K3S_NODE_NAME=master-2 K3S_TOKEN=<token> sh -s - server --server https://<ip-master1>:6443 --data-dir=/data/k3s
  kubectl get nodes -o wide
  ```
- Master-3 Node **Run on VM Master-3**
  ```bash
  curl -sfL https://get.k3s.io | K3S_NODE_NAME=master-3 K3S_TOKEN=<token> sh -s - server --server https://<ip-master1>:6443 --data-dir=/data/k3s
  kubectl get nodes -o wide
  ```

- Worker-1 Node (not yet add worker)
  **Run on VM Worker**
  ```bash
  curl -sfL https://get.k3s.io | K3S_NODE_NAME=worker-1 K3S_TOKEN=<token> sh -s - agent --server https://<ip-master1>:6443 --data-dir=/data/k3s
  ```

- Install add on Helm
  **Run on VM Master**
  ```bash
  curl -O https://get.helm.sh/helm-v3.13.0-linux-amd64.tar.gz
  tar -xzf helm-v3.13.0-linux-amd64.tar.gz
  mv linux-amd64/helm /usr/local/bin/helm
  helm version
  helm repo add stable https://charts.helm.sh/stable
  helm repo update
  ```
- Setup environment Master
  **Run on VM Master**
  ```bash
  vi ~/.bash_profile 
  export EDITOR=vim
  alias oc="kubectl"
  export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
  ```

- Verify
  ```bash
  k3s check-config
  ```

**Deploy application :**

> _K3s cli is using kubectl run from Master Node_

  
- Golang

  ```bash
  kubectl create namespace golang
  vi golang.yaml
  ```

  ```bash
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: golang
    namespace: golang
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: golang
    template:
      metadata:
        labels:
          app: golang
      spec:
        containers:
          - name: golang
            image: fransafu/simple-rest-golang:1.0.0
            resources:
              requests:
                memory: "64Mi"
                cpu: "100m"
              limits:
                memory: "128Mi"
                cpu: "500m"
            ports:
            - containerPort: 8080
            imagePullPolicy: Always
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: golang
    namespace: golang
  spec:
    ports:
    - port: 80
      targetPort: 8080
      name: tcp
    selector:
      app: golang
  ---
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: golang
    namespace: golang
    annotations:
      ingress.kubernetes.io/ssl-redirect: "false"
      kubernetes.io/ingress.class: "traefik"
  spec:
    ingressClassName: traefik
    rules:
    - http:
        paths:
        - backend:
            service:
              name: golang
              port:
                number: 80
          path: /
          pathType: Prefix
  ```
  
  > if you want to create different path than "/" example like "/golang" please use this below
  ```bash
  ---
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: golang-middleware
    namespace: golang
  spec:
    stripPrefix:
      prefixes:
        - "/golang"
      forceSlash: false
  ---
  apiVersion: traefik.containo.us/v1alpha1
  kind: IngressRoute
  metadata:
    name: golang
    namespace: golang
  spec:
    entryPoints:
      - web
    routes:
    - kind: Rule
      match: Host(`k3s.local`) && PathPrefix(`/golang`)
      priority: 10
      middlewares:
      - name: golang-middleware
        namespace: golang
      services:
      - kind: Service
        name: golang
        namespace: golang
        passHostHeader: true
        port: 80
  ```
  ```bash
  kubectl apply -f golang.yaml
  kubectl get ingress,all -n golang
  ```
- Longhorn
  ```bash
  vi values.yaml
  ---
  defaultSettings:
    backupTarget: s3://sentry@us-east-1/
    backupTargetCredentialSecret: uss-secret
    createDefaultDiskLabeledNodes: true
    defaultDataPath: /data/object-storage/
    replicaSoftAntiAffinity: false
    storageOverProvisioningPercentage: 600
    storageMinimalAvailablePercentage: 15
    upgradeChecker: false
    defaultReplicaCount: 2
    defaultDataLocality: disabled
    defaultLonghornStaticStorageClass: longhorn-static-example
    backupstorePollInterval: 500
    priorityClass: high-priority
    autoSalvage: false
    disableSchedulingOnCordonedNode: false
    replicaZoneSoftAntiAffinity: false
    volumeAttachmentRecoveryPolicy: never
    nodeDownPodDeletionPolicy: do-nothing
    guaranteedInstanceManagerCpu: 15
    orphanAutoDeletion: false
  ```
  ```bash
  helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --values values.yaml
  ```

  run on every node
  ```bash
  mkdir /data/object-storage
  ```
  setting internal svc for longhorn ui
  ```bash
  vi longhorn-internal-svc.yaml
  ---
  kind: Service
  apiVersion: v1
  metadata:
    name: longhorn-int-svc
    namespace: longhorn-system
  spec:
    type: ClusterIP
    selector:
      app: longhorn-ui
    ports:
    - name: http
      port: 8000
      protocol: TCP
      targetPort: 8000
  ```

  setting ingressroute & middleware
  ```bash
  vi longhorn-IngressRoute-traefik.yaml
  ---
  apiVersion: traefik.containo.us/v1alpha1
  kind: IngressRoute
  metadata:
    name: longhorn-ing-traefik
    namespace: longhorn-system
    annotations:
      ingress.kubernetes.io/ssl-redirect: "false"
      kubernetes.io/ingress.class: "traefik"

  spec:
    entryPoints:
      - web
    routes:
      - match: Host(`k3s.games.shopee.io`) && PathPrefix(`/longhorn`)
        kind: Rule
        services:
          - name: longhorn-int-svc
            port: http
        middlewares:
          - name: longhorn-add-trailing-slash
          - name: longhorn-stripprefix

  ---
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: longhorn-add-trailing-slash
    namespace: longhorn-system
  spec:
    redirectRegex:
      regex: ^.*/longhorn$
      replacement: /longhorn/

  ---
  apiVersion: traefik.containo.us/v1alpha1
  kind: Middleware
  metadata:
    name: longhorn-stripprefix
    namespace: longhorn-system
  spec:
    stripPrefix:
      prefixes:
        - /longhorn
  ```
  ```bash
  oc apply -f longhorn-internal-svc.yaml
  oc apply -f longhorn-IngressRoute-traefik.yaml
  ```
