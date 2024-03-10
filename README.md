In this post, we'll embark on installing [Appwrite](https://appwrite.io/), an open-source platform designed to facilitate the integration of authentication, databases, functions, and storage, enabling the development of scalable applications within our HomeLab setup.

# Prepartion

Referencing my [previous K3s setup post](https://seehiong.github.io/2023/setting-up-k3s/), let's initiate the installation process by deploying K3s server, this time with Traefik disabled:

```bash
sudo curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" K3S_KUBECONFIG_MODE="644" sh -
```

As usual, we verify the status with:

```bash
kc get no
```

Following that, we deploy Portainer to the K3s cluster:

```bash
mkdir portainer
cd portainer

# Download Portainer and deploy it
wget https://raw.githubusercontent.com/portainer/k8s/master/deploy/manifests/portainer/portainer.yaml -O deploy.yaml
kca deploy.yaml

# Check the port to use
kc get svc portainer -n portainer
```

Navigate to the specified port (e.g., http://bee:30777) and sets portainer up. and set up Portainer. Next, proceed by clicking on **Get started**.

![portainer-get-started](https://seehiong.github.io/images/appwrite/portainer-get-started.png)

## Generating Appwrite K8s Files

Referencing Appwrite's [Manual (Docker Compose)](https://appwrite.io/docs/advanced/self-hosting#manual), let's download the *docker-compose.yml* and *.env* files. We'll utilize [Kompose](https://kompose.io/) to convert them from Docker Compose to Kubernetes.

```bash
# Download Appwrite files
wget https://appwrite.io/install/compose -O docker-compose.yml
wget https://appwrite.io/install/env -O env

curl -L https://github.com/kubernetes/kompose/releases/download/v1.32.0/kompose-linux-amd64 -o kompose
chmod +x kompose

# Convert to K8s files
./kompose convert
```

![appwrite-converted-k8s-files](https://seehiong.github.io/images/appwrite/appwrite-converted-k8s-files.png)

## Preparing Appwrite Volumes

From my [previous NFS post](https://seehiong.github.io/2024/integrating-nfs-for-improved-scalability/), let's prepare the files to be placed in */var/lib/rancher/k3s/server/manifests*. We require a total of 9 storage classes. Refer to my public repository, [appwrite-k8s-deployment](https://github.com/seehiong/appwrite-k8s-deployment) for all necessary Kubernetes files mentioned in this post.

Below is an example for *appwrite-builds.yaml*:

```yaml
# Example: appwrite-builds.yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: appwrite-builds
  namespace: appwrite
spec:
  chart: nfs-subdir-external-provisioner
  repo: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
  set:
    nfs.server: tnas.local
    nfs.path: /mnt/public/appwrite
    storageClass.name: appwrite-builds
    storageClass.reclaimPolicy: Retain
    storageClass.accessModes: ReadWriteMany
    nfs.reclaimPolicy: Retain
```

Let's proceed to copying these 9 files to the auto-deployed folder:

```bash
ls appwrite-*.yaml
sudo cp appwrite-*.yaml /var/lib/rancher/k3s/server/manifests

# Verify files are copied correctly
sudo ls /var/lib/rancher/k3s/server/manifests
```

Similarly, create the corresponding persistent volume claims and apply them:

```bash
kc create ns appwrite
kca pvc-appwrite-builds.yaml 
kca pvc-appwrite-cache.yaml 
kca pvc-appwrite-certificates.yaml
kca pvc-appwrite-config.yaml
kca pvc-appwrite-functions.yaml
kca pvc-appwrite-influxdb.yaml
kca pvc-appwrite-mariadb.yaml
kca pvc-appwrite-redis.yaml 
kca pvc-appwrite-uploads.yaml
```

![appwrite-required-volumes](https://seehiong.github.io/images/appwrite/appwrite-required-volumes.png)

## Preparing Misc Services

As Kompose do not include any required environment variables, we'll manually map them against the original Docker Compose files and merge values from the .env file. These are the items to note of:

1. To add env var to all deployment files:

```yaml
# Example: mariadb-deployment.yaml
spec:
  template:
    spec:
      containers:
        - args:
            - mysqld
            - --innodb-flush-method=fsync
          # To include the required env
          env:
            - name: MYSQL_DATABASE
              value: appwrite
            - name: MYSQL_PASSWORD
              value: password
            - name: MYSQL_ROOT_PASSWORD
              value: rootsecretpassword
            - name: MYSQL_USER
              value: user
          image: mariadb:10.7
          name: appwrite-mariadb     
```

2. To use hostPath to access host's */var/run/docker.sock*:

```yaml
# Example: openruntimes-executor-deployment.yaml
spec:
  template:
    metadata:
    spec:
      hostname: appwrite-executor
      restartPolicy: Always
      volumes:
        # To use hostPath to host's /var/run/docker.sock and tmp folder
        - name: openruntimes-executor-claim0
          hostPath:
            path: /var/run/docker.sock
        - name: appwrite-builds
          persistentVolumeClaim:
            claimName: appwrite-builds
        - name: appwrite-functions
          persistentVolumeClaim:
            claimName: appwrite-functions
        - name: openruntimes-executor-claim3
          hostPath:
            path: /tmp
```

3. To use the fully qualified name for the DB and other services.

```yaml
# Example: appwrite-deployment.yaml
spec:
  template:
    spec:
      containers:
        - image: appwrite/appwrite:1.4.13
          name: appwrite
          # To include the required env
          env:
          # Using the fully qualified name to DB host
          - name: _APP_DB_HOST
            value: mariadb-svc.appwrite.svc
```

Apply all necessary DB and services:

```bash
# Deployments
kca influxdb-deployment.yaml
kca mariadb-deployment.yaml
kca openruntimes-executor-deployment.yaml
kca redis-deployment.yaml
kca telegraf-deployment.yaml

# Services
kca influxdb-svc.yaml
kca mariadb-svc.yaml
kca openruntimes-executor-svc.yaml
kca redis-svc.yaml
kca telegraf-svc.yaml
```

![appwrite-used-volumes](https://seehiong.github.io/images/appwrite/appwrite-used-volumes.png)

![appwrite-applications](https://seehiong.github.io/images/appwrite/appwrite-applications.png)

## Installing Appwrite Services

Finally, apply the remaining items:

```bash
# Deployments
kca appwrite-worker-webhooks-deployment.yaml
kca appwrite-worker-migrations-deployment.yaml
kca appwrite-worker-messaging-deployment.yaml
kca appwrite-worker-mails-deployment.yaml
kca appwrite-worker-functions-deployment.yaml

kca appwrite-worker-deletes-deployment.yaml
kca appwrite-worker-databases-deployment.yaml
kca appwrite-worker-certificates-deployment.yaml
kca appwrite-worker-builds-deployment.yaml
kca appwrite-worker-audits-deployment.yaml

kca appwrite-usage-deployment.yaml
kca appwrite-schedule-deployment.yaml
kca appwrite-realtime-deployment.yaml
kca appwrite-maintenance-deployment.yaml
kca appwrite-deployment.yaml

# You need to enter your own OPENAI API key for this to work
kca appwrite-assistant-deployment.yaml
```

Upon completion, all services should start normally:

![appwrite-applications-2](https://seehiong.github.io/images/appwrite/appwrite-applications-2.png)

![appwrite-used-volumes-2](https://seehiong.github.io/images/appwrite/appwrite-used-volumes-2.png)

Additionally, apply the *appwrite-svc.yaml*:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: appwrite-svc
  namespace: appwrite
spec:
  selector: 
    io.kompose.service: appwrite
  type: LoadBalancer
  ports:
  - name: appwrite-port
    protocol: TCP
    port: 80
    targetPort: 80
```

Then apply it:

```bash
kca appwrite-svc.yaml

# Check external-ip being assigned
kc get svc -n appwrite
```

Congratulations! By now, you should see the Appwrite login page:

![appwrite-login](https://seehiong.github.io/images/appwrite/appwrite-login.png)

![appwrite-hello-world-app](https://seehiong.github.io/images/appwrite/appwrite-hello-world-app.png)

That's all for this post! Happy Appwriting!

## Optional

### Enable Traefik

For those who wish to enable *traefik*, install k3s with this:

```bash
sudo curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -
```

Proceed with the rest of this post but to apply the following (instead of *appwrite-svc.yaml*):

```bash
kca appwrite-svc-with-ing.yaml
kca appwrite-ing.yaml

# Check your traefik loadbalancer external-ip
kc get svc -n kube-system
```

### Appwrite Functions

Update your deployment files accordingly for [Configuring Appwrite Functions with K3s](https://seehiong.github.io/2024/configuring-appwrite-functions-with-k3s/)