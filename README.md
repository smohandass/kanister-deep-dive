# kanister-deep-dive

This repo contains examples used for Kanister deep dive workshop

## Pre-reqs

1. Kuberenetes cluster that is running v1.16 or higher. 
2. Bastion server with kubectl, helm and docker installed
3. Access to an S3 bucket and credentials.
4. Install Kanister toold
5. A running Kanister controller
6. Database service 

## Setting up Pre-Reqs
Step 1-3 in the Pre-reqs is self explanatory and I am not covering here

### Install Kanister Tools

Kanister reporsitory provides two command-line tools kanctl and kando. kanctl simplifies the process of creating the custom kanister resources. Kando is a tool to simplify object store interactions from within blueprints. Installation of these tools requires Go binary 

Instructions to install Go binary can be found [here](https://go.dev/doc/install)

Instructions to install kanctl and kando can be found [here](https://docs.kanister.io/tooling.html?highlight=kando#install-the-tools)

### Install Kanister Operator on the k8s cluster

```
kubectl create ns kanister
helm repo add kanister https://charts.kanister.io/
helm -n kanister upgrade --install kanister --create-namespace kanister/kanister-operator
```

Verify that `kanister-operator` is successfully running 

```
kubectl get pods -n kanister
```

You should now see that Kanister has created four custom resources 
```
kubectl api-resources | grep kanister
actionsets                                                                                                                   cr.kanister.io/v1alpha1                       true         ActionSet
blueprints                                                                                                                   cr.kanister.io/v1alpha1                       true         Blueprint
profiles                                                                                                                     cr.kanister.io/v1alpha1                       true         Profile
repositoryservers                                                                                                            cr.kanister.io/v1alpha1                       true         RepositoryServer
```

### Setup a Database service

To work with the kanister examples oulined in this repo we need a database service. To keep it simple, let's install a bitnami mysql database service. 

```
kubectl create ns mysql
helm install mysql -n mysql oci://registry-1.docker.io/bitnamicharts/mysql
```
Verify that `mysql-0` pod is successfully running on the `mysql` namespace.

Use the following command to run a mysql-client pod that can be used to connect to the database to run queries and to create database objects. 

```
kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.36-debian-12-r10 --namespace mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash
mysql -h mysql.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

====================================================
# Example 1 : echo "Hello world" 

## Step 1: Create our first Blueprint


kubectl exec -it --namespace mysql mysql-0 -c mysql -- echo " Hello world - Example 1 "

```
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
  namespace: kanister
actions:
  backup:
    phases:
    - func: KubeExec
      name: mysqldump
      args:
        namespace: mysql
        pod: mysql-0
        container: mysql
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            echo " Hello world - Example 1 "
```

## Step 2: Create an Actionset to run the blueprint

```
apiVersion: cr.kanister.io/v1alpha1
kind: ActionSet
metadata:
  generateName: mysql-blueprint-backup-
  namespace: kanister
spec:
  actions:
  - name: backup
    blueprint: mysql-blueprint
    object:
      kind: StatefulSet
      name: mysql
      namespace: mysql
```

====================================================
# Example 2 : Introduce go template options


oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml


```
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
  namespace: kanister
actions:
  backup:
    phases:
    - func: KubeExec
      name: mysqldump
      args:
        namespace: '{{ .StatefulSet.Namespace }}'  
        pod: '{{ index .StatefulSet.Pods 0 }}'
        container: '{{ .StatefulSet.Namespace }}'
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            echo "Hello world - Example 2 "
            echo "Statefulset Name is : '{{ .StatefulSet.Namespace }}'" 
            echo "Replica Count is : '{{.Object.spec.replicas }}'"
```
Apply the yaml file

```
oc create -f mysql-blueprint.yaml
```
Create the actionset 
```
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql 
```
====================================================
# Example 3 : KubeExec with secret loading

oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml

apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
  namespace: kanister
actions:
  backup:
    phases:
    - func: KubeExec
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        namespace: '{{ .StatefulSet.Namespace }}'  
        pod: '{{ index .StatefulSet.Pods 0 }}'
        container: '{{ .StatefulSet.Namespace }}'
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            echo "Hello world - Example 3 "
            echo "Statefulset Name is : '{{ .StatefulSet.Namespace }}'" 
            echo "root_password : '{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}'"


oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql 

====================================================
# Example 4 : Introduce KubeTask

oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml
vi mysql-blueprint.yaml

apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: docker.io/bitnami/mysql:8.0.36-debian-12-r10
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            echo "Hello world - Example 4 "
            echo "Statefulset Name is : '{{ .StatefulSet.Namespace }}'" 
            echo "root_password : '{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}'"
            sleep 180


oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql 


====================================================
# Example 5 : Mysqldump of the database

oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml

apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: docker.io/bitnami/mysql:8.0.36-debian-12-r10
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            backup_file_path="/tmp/mysqldump.sql"
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            mysqldump --column-statistics=0 -u root --password=${root_password} -h mysql.mysql.svc.cluster.local --single-transaction --all-databases > ${backup_file_path}
            ls -l ${backup_file_path}
            sleep 180


oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql

====================================================
# Example 6 : Mysql dump of the database (versioned) and output to another pvc


oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml


cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-mysql
  namespace: mysql
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: rook-ceph-block
EOF


apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    phases:
    - func: PrepareData
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: docker.io/bitnami/mysql:8.0.36-debian-12-r10
        namespace: "{{ .StatefulSet.Namespace }}"
        volumes:
          backup-mysql: "/backup-data"
        podOverride:
          securityContext:
            fsGroup: 1001  
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            backupTimestamp=$(date "+%d_%b_%Y_%I_%M_%S_%p_%Z")
            mysqldump --column-statistics=0 -u root --password=${root_password} -h mysql.mysql.svc.cluster.local --single-transaction --all-databases > /tmp/mysqldump-${backupTimestamp}.sql
            cp /tmp/mysqldump-${backupTimestamp}.sql /backup-data/
            sleep 300

oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql
oc get pods -n mysql

====================================================
# Example 7 : Build an image that use mysqldump and kando 


docker version
systemctl status docker


FROM registry.access.redhat.com/ubi9/ubi:9.3-1476 as builder

RUN dnf clean all && rm -rf /var/cache/dnf
RUN dnf -y upgrade

# Download the RPM file to avoid timeouts during install
RUN curl -LO https://dev.mysql.com/get/mysql80-community-release-el9-5.noarch.rpm

# Install from the local file
RUN dnf install -y mysql80-community-release-el9-5.noarch.rpm
RUN dnf install -y mysql-community-client
RUN dnf install -y vi
RUN dnf install -y wget 

# Install Kanister Tools 
RUN wget https://raw.githubusercontent.com/kanisterio/kanister/master/scripts/get.sh
RUN ln -s /usr/bin/sha256sum /usr/bin/shasum
RUN sed -i 's/shasum -a 256/shasum/g' get.sh
RUN sh get.sh

docker build . -t kanister-demo-mysql:1.0.0
docker images
docker run -d  kanister-demo-mysql:1.0.0 tail -f /dev/null
docker ps -a
docker exec -it 44524cc8b665  /bin/bash
docker tag kanister-demo-mysql:1.0.0 bullseie/kanister-demo-mysql:1.0.0
docker push bullseie/kanister-demo-mysql:1.0.0



====================================================
# Example 8 : Mysqldump of the database and export to object store using Kando

# Example 8.1 - Mysqldump of the database using our custom image

oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml


apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: bullseie/kanister-demo-mysql:1.0.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            backupTimestamp=$(date "+%d_%b_%Y_%I_%M_%S_%p_%Z")
            backup_file_path="/tmp/dump-${backupTimestamp}.sql"
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            mysqldump --column-statistics=0 -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }} --single-transaction --all-databases > ${backup_file_path}
            sleep 360



oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql 

--------------------

# Example : 8.2 - Extending 8.1 to push our backup s3 location using Kando

oc delete blueprint mysql-blueprint -n kanister
oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
rm -f mysql-blueprint.yaml

# create s3compliant profile 

kanctl create profile s3compliant --access-key "xxxxxx" \
	--secret-key "xxxxxx" \
	--bucket "test-bucket" --region "us-east-2" \
	--namespace mysql

secret 's3-secret-syp2mp' created
profile 's3-profile-j5ps9' created

oc get profiles.cr.kanister.io -n mysql

oc delete secret -n mysql s3-secret-syp2mp
oc delete profiles.cr.kanister.io -n mysql s3-profile-j5ps9


apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: bullseie/kanister-demo-mysql:1.0.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            backupTimestamp=$(date "+%d_%b_%Y_%I_%M_%S_%p_%Z")
            backup_path="mysql_dumps/{{ .StatefulSet.Namespace }}/dump-${backupTimestamp}.sql"
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            dump_cmd="mysqldump --column-statistics=0 -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }} --single-transaction --all-databases"
            ${dump_cmd} | kando location push --profile '{{ toJson .Profile }}' --path "${backup_path}" -

oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql --profile mysql/s3-profile-j5ps9

oc get pods -n mysql -w


====================================================
# Example 9 : mysql restore



oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
oc delete blueprint mysql-blueprint -n kanister
rm -f mysql-blueprint.yaml

apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    outputArtifacts:
      mysqlbackup:
        keyValue:
          backuppath: "{{ .Phases.mysqldump.Output.backuppath }}"
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: ghcr.io/kanisterio/mysql-sidecar:0.107.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            backupTimestamp=$(date "+%d_%b_%Y_%I_%M_%S_%p_%Z")
            backup_file_path="mysql_dumps/{{ .StatefulSet.Namespace }}/dump-${backupTimestamp}.sql"
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            dump_cmd="mysqldump --column-statistics=0 -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }} --single-transaction --all-databases"
            ${dump_cmd} | kando location push --profile '{{ toJson .Profile }}' --path "${backup_file_path}" -
            kando output backuppath "${backup_file_path}"
  restore:
    inputArtifactNames:
    - mysqlbackup
    phases:
    - func: KubeTask
      name: mysqlrestore
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: ghcr.io/kanisterio/mysql-sidecar:0.107.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          backup_file_path='{{ .ArtifactsIn.mysqlbackup.KeyValue.backuppath }}'
          echo "backup_file_path is : ${backup_file_path}"
          root_password="{{ index .Phases.mysqlrestore.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
          restore_cmd="mysql -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }}"
          kando location pull --profile '{{ toJson .Profile }}' --path "${backup_file_path}" - | ${restore_cmd}




oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql --profile mysql/s3-profile-j5ps9
oc get actionset -n kanister -w



kanctl create actionset --action restore --from "backup-rkbr2" --namespace kanister --blueprint mysql-blueprint --profile mysql/s3-profile-j5ps9 --statefulset mysql/mysql
oc get actionset -n kanister -w




====================================================
# Example 10 : Delete the backup in object store


oc get actionset --no-headers -n kanister | awk '{print $1}' | xargs oc delete actionset -n kanister
oc delete blueprint mysql-blueprint -n kanister
rm -f mysql-blueprint.yaml

apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mysql-blueprint
actions:
  backup:
    outputArtifacts:
      mysqlbackup:
        keyValue:
          backuppath: "{{ .Phases.mysqldump.Output.backuppath }}"
    phases:
    - func: KubeTask
      name: mysqldump
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: bullseie/kanister-demo-mysql:1.0.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
          - bash
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            backupTimestamp=$(date "+%d_%b_%Y_%I_%M_%S_%p_%Z")
            backup_file_path="mysql_dumps/{{ .StatefulSet.Namespace }}/dump-${backupTimestamp}.sql"
            root_password="{{ index .Phases.mysqldump.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
            dump_cmd="mysqldump --column-statistics=0 -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }} --single-transaction --all-databases"
            ${dump_cmd} | kando location push --profile '{{ toJson .Profile }}' --path "${backup_file_path}" -
            kando output backuppath "${backup_file_path}"
  restore:
    inputArtifactNames:
    - mysqlbackup
    phases:
    - func: KubeTask
      name: mysqlrestore
      objects:
        mysqlSecret:
          kind: Secret
          name: '{{ index .Object.metadata.labels "app.kubernetes.io/instance" }}'
          namespace: '{{ .StatefulSet.Namespace }}'
      args:
        image: bullseie/kanister-demo-mysql:1.0.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          backup_file_path='{{ .ArtifactsIn.mysqlbackup.KeyValue.backuppath }}'
          root_password="{{ index .Phases.mysqlrestore.Secrets.mysqlSecret.Data "mysql-root-password" | toString }}"
          restore_cmd="mysql -u root --password=${root_password} -h {{ index .Object.metadata.labels "app.kubernetes.io/instance" }}"
          kando location pull --profile '{{ toJson .Profile }}' --path "${backup_file_path}" - | ${restore_cmd}
  delete:
    inputArtifactNames:
    - mysqlBackup
    phases:
    - func: KubeTask
      name: mysqldelete
      args:
        image: bullseie/kanister-demo-mysql:1.0.0
        namespace: "{{ .StatefulSet.Namespace }}"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          backup_file_path='{{ .ArtifactsIn.mysqlbackup.KeyValue.backuppath }}'
          kando location delete --profile '{{ toJson .Profile }}' --path "${backup_file_path}"



oc create -f mysql-blueprint.yaml
kanctl create actionset --action backup --namespace kanister --blueprint mysql-blueprint --statefulset mysql/mysql --profile mysql/s3-profile-j5ps9
oc get actionset -n kanister -w



kanctl create actionset --action restore --from "backup-ssmqn" --namespace kanister --blueprint mysql-blueprint --profile mysql/s3-profile-j5ps9 --statefulset mysql/mysql
oc get actionset -n kanister -w

kanctl create actionset --action delete --from "backup-npsq8" --namespace kanister --blueprint mysql-blueprint --profile mysql/s3-profile-j5ps9 --statefulset mysql/mysql
oc get actionset -n kanister -w

