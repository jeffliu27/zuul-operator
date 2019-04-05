A Zuul Operator PoC
=======

## Requirements:

* [OKD](https://github.com/openshift/origin/releases/download/v3.11.0/openshift-origin-client-tools-v3.11.0-0cbc58b-linux-64bit.tar.gz)
* [SDK](https://github.com/operator-framework/operator-sdk#quick-start)
* [Zookeeper Operator](https://github.com/pravega/zookeeper-operator#install-the-operator)

## Prepare cluster

```shell
sudo -i oc cluster up
sudo chown root:fedora /var/run/docker.sock

oc login -u developer -p dev
docker login -u developer -p $(oc whoami -t) $(oc registry info)

# Log as admin to install crd
sudo cat /root/.kube/config > ~/.kube/config
oc login -u system:admin
oc project default
```

## Install Postgress Operator

Follow [install instruction](https://crunchydata.github.io/postgres-operator/stable/installation/),
basically:
```
vi ./pv/crunchy-pv.json  # set volume size and pv number
oc apply -f ./pv/crunchy-pv.json
oc apply -f ./deploy/cluster-rbac.yaml
oc apply -f ./deploy/rbac.yaml
./deploy/deploy.sh
```

## Install Zookeeper Operator

```shell
oc create -f https://raw.githubusercontent.com/pravega/zookeeper-operator/master/deploy/crds/zookeeper_v1beta1_zookeepercluster_crd.yaml
oc create -f https://raw.githubusercontent.com/pravega/zookeeper-operator/master/deploy/default_ns/rbac.yaml
oc create -f https://raw.githubusercontent.com/pravega/zookeeper-operator/master/deploy/default_ns/operator.yaml
```

## Install Zuul Operator

```shell
operator-sdk build 172.30.1.1:5000/myproject/zuul-operator:latest
docker push 172.30.1.1:5000/myproject/zuul-operator:latest

oc create -f deploy/crds/zuul-ci_v1alpha1_zuul_crd.yaml
oc create -f deploy/rbac.yaml
oc create -f deploy/operator.yaml
```

Look for operator pod and check it's output

```shell
$ oc get pods
NAME                            READY     STATUS    RESTARTS   AGE
zuul-operator-c64756f66-rbdmg   2/2       Running   0          3s
$ oc logs zuul-operator-c64756f66-rbdmg -c operator
...
{"level":"info","ts":1554197305.5853095,"logger":"cmd","msg":"Go Version: go1.10.3"}
{"level":"info","ts":1554197305.5854425,"logger":"cmd","msg":"Go OS/Arch: linux/amd64"}
{"level":"info","ts":1554197305.5854564,"logger":"cmd","msg":"Version of operator-sdk: v0.6.0"}
{"level":"info","ts":1554197305.5855,"logger":"cmd","msg":"Watching namespace.","Namespace":"default"}
...
```

## Usage

```
$ oc apply -f - <<EOF
apiVersion: zuul-ci.org/v1alpha1
kind: Zuul
metadata:
  name: example-zuul
spec:
  # Optional user-provided ssh key
  sshsecretename: ""
  merger:
    instances: 0
  executor:
    instances: 1
  web:
    instances: 1
  connections: []
  tenants:
    - tenant:
        name: demo
        source: {}
EOF
zuul.zuul-ci.org/example-zuul created

$ oc get zuul
NAME           AGE
example-zuul   10s

# Get zuul public key
$ oc get secret example-ssh-secret-pub -o "jsonpath={.data.id_rsa\.pub}" | base64 -d
ssh-rsa AAAAB3Nza...

$ oc get pods
NAME                                      READY     STATUS      RESTARTS   AGE
example-zuul-executor-696f969c4-6cpjv     1/1       Running     0          8s
example-zuul-pg-5dfc477bff-8426l          1/1       Running     0          30s
example-zuul-scheduler-77b6cf7967-ksh64   1/1       Running     0          11s
example-zuul-web-5f744f89c9-qjp9l         1/1       Running     0          6s
example-zuul-zk-0                         1/1       Running     0          22s

$ oc get svc example-zuul-web
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                     PORT(S)                   AGE
example-zuul-web        ClusterIP      172.30.209.181   <none>                          80/TCP                    41s

$ curl 172.30.209.181/api/tenants
[{"name": "demo", "projects": 0, "queue": 0}]
```
