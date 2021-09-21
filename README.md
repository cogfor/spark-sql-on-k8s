# spark-sql-on-k8s

The simplest spark sql on kubernetes production environment deployment plan

## Architecture and design

For detailed architecture and design content, please refer to https://zhuanlan.zhihu.com/p/345214051

## Background

Spark SQL Server, which is highly efficient, available for production, and supports rapid deployment, does not have a good solution. The native Spark Thrift Server cannot solve the multi-tenant problem very well. The implementation is very simple. It provides the external thrift interface and realizes the processing of spark sql by sharing the spark session internally, which is not suitable for use in a production environment.

Kyuubi provides a better Spark SQL multi-tenant implementation solution. Through the timeout recovery of the SQL engine, there is also a better balance in performance. However, kyuubi is not simple enough to install and deploy.

Spark-sql-on-k8s hopes to provide as simple as installing and deploying mysql or postgresql to use spark sql. Provide users with a standard jdbc interface, which has complete spark capabilities, such as support for UDF, machine learning, etc. Allow spark users to only focus on sql, without having a big data foundation, but also to enjoy the experience brought by big data technology.

Based on spark implementation, spark-sql-on-k8s can be easily extended to support data lakes, such as deta lake or iceberg. It can also easily support ETL, BI and other application scenarios. The characteristics of fast, simple, convenient, and functionally complete are highly recommended for the use experience of small and medium-sized clusters.

## Minimal dependencies
The minimum set of dependencies for spark-sql-on-k8s installation and deployment: kubernetes, mysql (postgresql), S3 compatible storage. In addition to some quick installation tools for the installation and deployment of kubernetes, there are also some simplified (fully compatible) implementation solutions (such as k3s, k0s, etc.). These standard simplified versions are very suitable for small and medium-sized clusters, and there is no difference in use, but Installation, deployment, operation and maintenance are much simplified. The application of mysql (postgresql) is of course very extensive, basically in any cluster, there is installation and deployment, so the requirements for mysql, in fact, there is very little work to be done. S3 compatible storage is preferred over HDFS for several reasons: One is to simplify the difficulty of O&M deployment. For small and medium-sized clusters, HDFS has no advantage in cost performance compared to S3, and O&M is more complicated; the other is to embrace the cloud Native, there are many existing S3 compatible cloud storage providers, and they also have certain advantages in terms of usage cost. Of course, you can also choose minio to build an S3-compatible storage layer (free but requires operation and maintenance support).

Of course, HDFS and hive metastore are also fully supported and used in a standard Hadoop environment. And can support kerberos authentication method (depending on kyuubi support).

### Open source dependency

1. spark-operator-on-k8s（spark application controller）

https://github.com/GoogleCloudPlatform/spark-on-k8s-operator

Kubernetes operator for managing the lifecycle of Apache Spark applications on Kubernetes.

2. Forked kyuubi (jdbc server and sql engine, mainly adding k8s SessionManager and K8s Session Impl functions to realize spark application management capabilities based on spark operator). The following are the open source kyuubi and forked kyuubi paths.

https://github.com/yaooqinn/kyuubi

https://github.com/yilong2001/kyuubi

Kyuubi is a high-performance universal JDBC and SQL execution engine, built on top ofApache Spark. The goal of Kyuubi is to facilitate users to handle big data like ordinary data.


## Install：

```shell
git clone --recursive https://github.com/yilong2001/spark-sql-on-k8s.git

```

### create namespaces (k8s namespace)

```shell
kubectl create namespace spark-operator
kubectl create namespace spark-jobs
kubectl create namespace spark-history
```

### install traefik v2 crds

Note: not required when using k3d
 
```
kubectl apply -f traefik-helm-chart/traefik/crds/
```

### install traefik v2 ingress resources。

```shell
First, modify loadbalance external ip.
If not set, use any ip in the k8s cluster.

File path:：k8s/deploy/traefik-v2/004-service.yaml

  externalIPs:
    - xxx.xxx.xxx.xxx

kubectl apply -f k8s/deploy/traefik-v2/
```


### Install spark operator on k8s (apply spark-operator)

```shell
kubectl apply -f k8s/deploy/spark-operator
```

### Build spark sql on k8s (apply sparkop-ctrl)

```shell
Optionally: set the image builder (docker / img / ... ) in the make script

sh make.sh 
```

Before creating the control, the parameters that need to be modified are as follows (depending on s3 and mysql):

```shell
Modify the configuration parameters in `k8s/deploy/sparkop-ctrl/sparkop-deployment.yml`:

S3-compatible storage can be object storage on public clouds; or use minio to build object storage services by yourself.

--s3-upload-dir=s3://testbucket
--s3-endpoint=192.168.42.1:9000
--s3-accesskey=minioadmin
--s3-secretkey=minioadmin

Metadatabase, that is, spark sql metastore database, the default database name is hive, please create the hive database in advance; otherwise, you need to configure a user with the permission to create the database. Currently only supports mysql database.

--metadb-host=192.168.42.1
--metadb-user=xxx
--metadb-pw=xxx
```

### Deploy spark

```shell
kubectl apply -f k8s/deploy/spark/
```

### Deploy kyuubi server

```shell
kubectl apply -f k8s/deploy/kyuubi-server/
```


## jdbc connect

```shell
The jdbc connection can be accompanied by spark operating parameters to set operating parameters such as memory and cpu core.

For example:

jdbc:hive2://spark-server-jdbc-ingress:10019/?spark.dynamicAllocation.enabled=true;spark.dynamicAllocation.maxExecutors=500;spark.shuffle.service.enabled=true;spark.executor.cores=3;spark.executor.memory=2g
```

## Spark ui view SQL details

```shell
Visit the following URL to view the history of spark sql under a certain user: 
http://spark.mydomain.io/proxy/spark/spark-jobs/spark-sql-${user}

For example, if you log in as the admin user, you can access the following URL:
http://spark.mydomain.io/proxy/spark/spark-jobs/spark-sql-admin
```

