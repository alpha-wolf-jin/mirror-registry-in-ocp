# mirror-registry-in-ocp

In the disconnection environemnt, we need one mirroring registry. 

But we cannot found extra VM with enough disk space to create mirroring registry and port 5000 in bastion is blocked in lab environment

Here, we create one serice in OCP for mirroring registry.

## update git

```
echo "# mirror-registry-in-ocp" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/mirror-registry-in-ocp.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main


git add . ; git commit -a -m "update README" ; git push -u origin main
```

## Prepare the images

```
[lab-user@bastion registry]$ podman pull docker.io/library/registry:2.7.1

[lab-user@bastion registry]$ oc whoami --show-server
https://api.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com:6443

[lab-user@bastion registry]$ oc login -u opentlc-mgr https://api.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com:6443

[lab-user@bastion registry]$ podman login -u opentlc-mgr -p $(oc whoami --show-token) default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com

[lab-user@bastion registry]$ podman tag b8604a3fe854 default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/docker-registry/registry:2.7.1

[lab-user@bastion registry]$ podman push b8604a3fe854 default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/docker-registry/registry:2.7.1

```
# Create docker registry

>Note: below hostanme has to changed  accordingly when using differnt names
```
        - name: REGISTRY_OPENSHIFT_SERVER_ADDR
          value: 'docker-registry.docker-registry.svc.cluster.local:5000'
```

```
[lab-user@bastion registry]$ oc adm new-project docker-registry

[lab-user@bastion registry]$ oc project docker-registry

[lab-user@bastion registry]$ htpasswd -bBc ./htpasswd admin redhat123

[lab-user@bastion registry]$ oc create secret generic auth-secret --from-file=htpasswd=./htpasswd

[lab-user@bastion registry]$ vi pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: docker-registry-data-01
  namespace: docker-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
  storageClassName: gp2
  volumeMode: Filesystem

[lab-user@bastion registry]$ oc apply -f pvc.yaml 

[lab-user@bastion registry]$ oc get pvc
NAME                      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
docker-registry-data-01   Pending                                      gp2            23s

[lab-user@bastion registry]$ cat deploy-docker-registry.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: docker-registry
    app.kubernetes.io/component: docker-registry
    app.kubernetes.io/instance: docker-registry
  name: docker-registry
  namespace: docker-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      deployment: docker-registry
  template:
    metadata:
      labels:
        deployment: docker-registry
    spec:
      containers:
      - env:
        - name: REGISTRY_AUTH
          value: "htpasswd"
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: "Registry Realm"
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: "/auth/htpasswd"
        - name: REGISTRY_OPENSHIFT_SERVER_ADDR
          value: 'docker-registry.docker-registry.svc.cluster.local:5000'
        - name: REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED
          value: 'true'
        image: docker.io/library/registry:2.7.1
        imagePullPolicy: Always
        name: docker-registry
        ports:
        - containerPort: 5000
          protocol: TCP
        resources: {}
        volumeMounts:
        - mountPath: "/registry" 
          name: mypd 
        - name: auth-vol
          mountPath: "/auth/htpasswd"
          readOnly: true
          subPath: htpasswd
      volumes:
      - name: mypd
        persistentVolumeClaim:
          claimName: docker-registry-data-01
      - name: auth-vol
        secret:
          secretName: auth-secret

[lab-user@bastion registry]$ oc apply -f deploy-docker-registry.yaml  

[lab-user@bastion registry]$ oc get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/docker-registry-66545c65f5-2fkmn   1/1     Running   0          7m20s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/docker-registry   1/1     1            1           7m20s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/docker-registry-66545c65f5   1         1         1       7m20s

[lab-user@bastion registry]$ oc expose deploy docker-registry

[lab-user@bastion registry]$ oc create route edge --service=docker-registry

[lab-user@bastion registry]$ oc get route
NAME              HOST/PORT                                                                          PATH   SERVICES          PORT    TERMINATION   WILDCARD
docker-registry   docker-registry-docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com          docker-registry   <all>   edge          None
[lab-user@bastion registry]$ 

[lab-user@bastion registry]$ curl -u admin:redhat123 https://docker-registry-docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/v2/_catalog 
{"repositories":[]}

```

**Change to short name**

```
[lab-user@bastion registry]$ oc edit route docker-registry
. . .
spec:
  host: docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com
. . .

[lab-user@bastion registry]$ oc get route
NAME              HOST/PORT                                                          PATH   SERVICES          PORT    TERMINATION   WILDCARD
docker-registry   docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com          docker-registry   <all>   edge          None

[lab-user@bastion registry]$ curl -u admin:redhat123 https://docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/v2/_catalog
{"repositories":[]}

```




## Prepare the images for disconnection environment

### Prepare the docker image

Transfer dock image file into disconnectin environemnt
> Use the podman pull ... ; podman save .. to tar file ; podman load -i to load in disconencted env

here, we directly use podman pull
```
[lab-user@bastion registry]$ podman pull docker.io/library/registry:2.7.1

[lab-user@bastion registry]$ podman images
REPOSITORY                                                                                                         TAG     IMAGE ID       CREATED        SIZE
docker.io/library/registry                                                                                         2.7.1   b8604a3fe854   5 months ago   26.8 MB

```



## Push the image to the internal registry

```

[lab-user@bastion registry]$ oc whoami --show-server
https://api.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com:6443

[lab-user@bastion registry]$ oc login -u opentlc-mgr https://api.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com:6443

[lab-user@bastion registry]$ podman login -u opentlc-mgr -p $(oc whoami --show-token) default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com

[lab-user@bastion registry]$ podman tag b8604a3fe854 default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/docker-registry/registry:2.7.1

[lab-user@bastion registry]$ podman images
REPOSITORY                                                                                                         TAG     IMAGE ID       CREATED        SIZE
docker.io/library/registry                                                                                         2.7.1   b8604a3fe854   5 months ago   26.8 MB
default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/docker-registry/registry   2.7.1   b8604a3fe854   5 months ago   26.8 MB

[lab-user@bastion registry]$ podman push default-route-openshift-image-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/docker-registry/registry:2.7.1
```


### Change the image in deploy conf
**from**
- `docker.io/library/registry:2.7.1`
**to**
- `image-registry.openshift-image-registry.svc.cluster.local:5000/docker-registry/registry:2.7.1`


```
[lab-user@bastion registry]$ oc eidt deploy docker-registry
...
    spec:
      containers:
      - env:
        - name: REGISTRY_AUTH
          value: htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: Registry Realm
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/htpasswd
        - name: REGISTRY_OPENSHIFT_SERVER_ADDR
          value: docker-registry.docker-registry.svc.cluster.local:5000
        - name: REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED
          value: "true"
        image: docker.io/library/registry:2.7.1
        image: image-registry.openshift-image-registry.svc.cluster.local:5000/docker-registry/registry:2.7.1
...

[lab-user@bastion registry]$ oc get pod
NAME                               READY   STATUS    RESTARTS   AGE
docker-registry-6655c7d757-kfqnz   1/1     Running   0          12s

[lab-user@bastion registry]$ curl -u admin:redhat123 https://docker-registry.apps.cluster-n2p5z.n2p5z.sandbox1445.opentlc.com/v2/_catalog
{"repositories":[]}

```
