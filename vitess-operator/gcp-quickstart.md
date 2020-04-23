# Getting started with the Vitess Operator on Google Cloud Platform

## Introduction

This document shows how to use the Vitess Operator to deploy a Vitess cluster on Google Cloud Platform.

## Prerequisites

This guide assumes you have the following components and services in place: <!-- what? -->

+ A [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs) (GKE) cluster;
+ A local `kubectl` client [configured to access your GKE cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl);

<!--
+
+ A GCS storage bucket (https://cloud.google.com/storage/docs/creating-buckets);
+ A GCP service account (https://cloud.google.com/storage/docs/projects#service-accounts) with access to the GCS storage bucket;
+ A Kubernetes secret matching your service account (https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform#step_3_create_service_account_credentials);

These prerequisites still necessite a step where the user updates the exampledb.yaml file to reflect the name of the K8s secret. Is this the best way to document this?-->
+ A local [installation of `vtctlclient`](https://vitess.io/docs/get-started/kubernetes/#prerequisites).

<!-- Should we do a version of this quickstart that omits operator backups? This would simplify it a lot. -->

## Overview

To deploy a Vitess cluster on GCP using the Vitess Operator, follow these steps:

1. Install the operator and example database configuration files. <!-- Reword? -->
1. Apply the operator configuration file against your Kubernetes cluster.
1. Apply the database configuration file to your cluster.
1. Port-forward the `vtctld` service to your Kubernetes cluster.
1. Apply the SQL schema to your Vitess database.
1. Expose the Vitess service.
1. Connect to your Vitess database using a MySQL client.

## Step 1. Install the operator.

Download the operator configuration file here:

[https://storage.googleapis.com/vitess-operator/install/operator.yaml]

Download the database configuration file here:

[https://storage.googleapis.com/vitess-operator/examples/exampledb.yaml]

## Step 2. Apply the operator configuration file against your Kubernetes cluster.

Enter the following command:

```sh
> kubectl apply -f operator.yaml
```

You should see the following output:

```sh
customresourcedefinition.apiextensions.k8s.io/etcdlockservers.planetscale.com created
1
customresourcedefinition.apiextensions.k8s.io/vitessbackups.planetscale.com created
customresourcedefinition.apiextensions.k8s.io/vitessbackupstorages.planetscale.com created
customresourcedefinition.apiextensions.k8s.io/vitesscells.planetscale.com created
customresourcedefinition.apiextensions.k8s.io/vitessclusters.planetscale.com created
customresourcedefinition.apiextensions.k8s.io/vitesskeyspaces.planetscale.com created
customresourcedefinition.apiextensions.k8s.io/vitessshards.planetscale.com created
serviceaccount/vitess-operator created
role.rbac.authorization.k8s.io/vitess-operator created
rolebinding.rbac.authorization.k8s.io/vitess-operator created
priorityclass.scheduling.k8s.io/vitess created
priorityclass.scheduling.k8s.io/vitess-operator-control-plane created
deployment.apps/vitess-operator created
```

You can verify the status of the operator pods using the following command:

```sh
> kubectl get pods
```

## Step 3. Apply the database configuration file to your cluster.

Apply the example database configuration to your Kubernetes cluster using the following command:

<!-- Where is the file? Why not just apply directly from URL? Removes two steps out of eight. -->

```sh
> kubectl apply -f exampledb.yaml
```

You should see the following output:

```sh
vitesscluster.planetscale.com/example created
secret/example-cluster-config created
```

After a few minutes, you should see the pods for your keyspace being created using this command:

```sh
> kubectl get pods
```

You should see output like this:

```sh
NAME READY STATUS RESTARTS AGE
example-90089e05-vitessbackupstorage-subcontroller 1/1 Running 0 59s
example-etcd-faf13de3-1 1/1 Running 0 59s
example-etcd-faf13de3-2 1/1 Running 0 59s
example-etcd-faf13de3-3 1/1 Running 0 59s
example-uscentral1a-vtctld-6a268099-56c48bbc89-6r9dp 1/1 Running 2 58s
example-uscentral1a-vtgate-bbffae2f-54d5fdd79-gmwlm 0/1 Running 2 54s
example-uscentral1a-vtgate-bbffae2f-54d5fdd79-jldzg 0/1 Running 2 54s
example-vttablet-uscentral1a-0261268656-d6078140 2/3 Running 2 58s
example-vttablet-uscentral1a-1579720563-f892b0e6 2/3 Running 2 59s
example-vttablet-uscentral1a-2253629440-17557ac0 2/3 Running 2 58s
example-vttablet-uscentral1a-3067826231-d454720e 2/3 Running 2 59s
example-vttablet-uscentral1a-3815197730-f3886a80 2/3 Running 2 58s
example-vttablet-uscentral1a-3876690474-0ed30664 2/3 Running 2 59s
vitess-operator-6f54958746-mr9hp 1/1 Running 0 17m
```

## Step 4. Port-forward the `vtctld` service to your Kubernetes cluster.

Use the following command:

```sh
kubectl port-forward --address localhost deployment/$(kubectl get deployment --selector="planetscale.com/component=vtctld" -o=jsonpath="{.items..metadata.name}") 15999:15999
```
You should now be able to see all of your tablets using the following command:

<!-- This step requires vtctlclient to be installed also. Add to prereqs? -->

```sh
> vtctlclient -server localhost:15999 ListAllTablets
```

You should see output like this:

```sh
uscentral1a-0261268656 main -80 replica 10.16.1.16:15000 10.16.1.16:3306 []
uscentral1a-1579720563 main 80- replica 10.16.1.15:15000 10.16.1.15:3306 []
uscentral1a-2253629440 main -80 replica 10.16.0.18:15000 10.16.0.18:3306 []
uscentral1a-3067826231 main 80- replica 10.16.0.17:15000 10.16.0.17:3306 []
uscentral1a-3815197730 main 80- master 10.16.2.20:15000 10.16.2.20:3306 []
uscentral1a-3876690474 main -80 master 10.16.2.21:15000 10.16.2.21:3306 []
```

## Step 5. Apply the VSchema to your Vitess database.

Download the example VSchema here:

[https://storage.googleapis.com/vitess-operator/examples/vschema.json]

Apply the VSchema using the following command:

```sh
vtctlclient -server localhost:15999 ApplyVSchema -vschema "$(cat ./vschema.json)" main
```

## Step 5. Apply the SQL schema to your Vitess database.

Apply the SQL schema using the following command:


<!-- Is this the right format to be using for shell commands? Also, this requires that the schema be located in the working directory. -->

```sh
> vtctlclient -server localhost:15999 ApplySchema -sql "$(cat ./schema.sql)" main
```

## Step 6. Expose the Vitess service.

Expose the service using the following command:

<!-- Breaking up this command into two parts. Try getting Jacque's embedded syntax to work.-->

The following command outputs the deployment name:

```sh
> kubectl get deployment --selector="planetscale.com/component=vtgate" -o=jsonpath="{.items..metadata.name}"
```

The following command exposes the service using the deployment name from above.

```sh
kubectl expose deployment [deployment_name] --type=LoadBalancer --name=test-vtgate --port 3306 --target-port 3306
```

You should see the following output:

```
service/test-vtgate exposed
```

## Step 7. Determine the external IP for the Load Balancer service

Use the following command to get the IP for your LoadBalancer service:

```sh
kubectl get service test-vtgate
```

You should see output like the following:


```sh
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
test-vtgate LoadBalancer 10.35.244.55 35.238.202.46 3306:32157/TCP 90s
```

## Step 7. Connect to your Vitess database using a MySQL client.                     

Use the IP from the previous step to connect to your Vitess database using a command like the following:

<!-- Is this the right format for CLI variables? -->

```sh
mysql -u user -h [external_ip]
```

For example, to connect to the example IP address from the previous step, use the following command:

```sh
> mysql -u user -h 35.238.202.46 main -p -A
```

You can now submit queries against your Vitess database from your MySQL client.

For example, the following query displays the tables in your database with VSchemas: <!-- What? -->

```sql
> SHOW VSCHEMA TABLES;
```

The above query should return the following output:

```sql
+----------------+
| Tables         |
+----------------+
| dual           |
| users          |
| users_name_idx |
+----------------+
3 rows in set (0.06 sec)
```

## Cleanup 

To clean up this quickstart, follow these steps:

1. Delete the service. <!-- What service? How? -->
1. Delete the database configuration file.
1. Delete the operator configuration file.
