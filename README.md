# mirror-registry-in-ocp

In the disconnection environemnt, we need one mirroring registry. 

But we cannot found extra VM to create mirroring registry and port 5000 in bastion is blocked. 

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

[lab-user@bastion registry]$ oc get pod
NAME                               READY   STATUS    RESTARTS   AGE
docker-registry-66545c65f5-2fkmn   1/1     Running   0          3m14s

```
