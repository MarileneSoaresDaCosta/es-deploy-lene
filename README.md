## Table of Contents

- [Development](#development)
- [Deployment](#deployment)
- [Useful Commands](#useful-commands)
- [References](#references)

## Development

Most of the configuration included here for minikube will also work for deploying ECK and Logstash to different environments such as staging and production.

minikube requires a wrapper to make the Elasticsearch cluster available with a LoadBalancer. See Justin Lim's [kube.sh](https://www.gooksu.com/2021/05/minikube-wrapper-kubernetes/)

```
chmod +x kube.sh
```

```
./kube.sh start # starts the minikube environment might prompt you for sudo password
```

Switch your current context to minikube

```
kubectl config use-context minikube
```

#### Elastic Cloud on Kubernetes (ECK)

Refer to this article for an in-depth walk-through of the process to get minikube environment up: [Elastic Cloud on kubernetes (ECK) on minikube | Justin Lim](https://www.gooksu.com/2021/05/elastic-cloud-on-kubernetes-eck-on-minikube/)

- Install the operator(version 1.3.2)
- Create a 1 node ES 7.9.0 cluster
- Find the PASSWORD for the elastic user
- Create kibana 7.9.0 to for the ES cluster
- Expand the cluster to 3 nodes
- Update elasticsearch license
- Upgrade elasticsearch to 7.10.2
- Upgrade kibana to 7.10.2
- Upgrade the operator to 1.5.0
- SSL configuration

Additional install/setup reference: [How to deploy the Elastic Stack on Kubernetes](https://raphaeldelio.medium.com/deploy-the-elastic-stack-in-kubernetes-with-the-elastic-cloud-on-kubernetes-eck-b51f667828f9)

#### Logstash

> Logstash doesn't come with ECK but can easily be added to the cluster to ingest and ship data to Elasticsearch. See this article for a walk-through: [How To Deploy Logstash and Filebeat On Kubernetes With ECK and SSL](https://raphaeldelio.medium.com/deploy-logstash-and-filebeat-on-kubernetes-with-eck-ssl-and-filebeat-d9f616737390)

```
kubectl apply -f logstash-cm.yaml
```

```
kubectl apply -f logstash-deploy.yaml
```

```
kubectl apply -f logstash-service.yaml
```

##### JDBC Input Plugin for Logstash

This plugin was created as a way to ingest data in any database with a JDBC interface into Logstash. You can periodically schedule ingestion using a cron syntax.

The jar file for the MySQL driver is included in the [image](dezrogers/logstash-mysql-jdbc:latest) built with a simple Dockerfile. Get the jar file for different DBMS or different version of MySQL [here](https://www.soapui.org/docs/jdbc/reference/jdbc-drivers/).

```Dockerfile
FROM docker.elastic.co/logstash/logstash:7.13.2

# copy sql jdbc jars
COPY ./mysql-connector-java-8.0.25.jar /usr/share/logstash/logstash-core/lib/jars/mysql-connector-java.jar
```

For more details on Jdbc input plugin like driver options and scheduling see the [docs](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)

- [Configuring multiple SQL statements](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html#_configuring_multiple_sql_statements)

#### MySQL

If using DB on localhost, i.e. not in a container:

- DB binding address in your `my.cnf` set to all IPs -> `0.0.0.0` for minikube host access to work properly
  - Note: mysql config gets added in different places depending on your installation method and OS. To find possible locations on MacOS, run `mysql --help | grep my.cnf`[3]
- DB user with username and password that's not null

```
mysql> CREATE USER 'test_user'@'localhost' IDENTIFIED WITH mysql_native_password BY '{superSecretPassword!123}';
mysql> GRANT ALL ON `testdb`.* TO 'test_user'@'localhost';
```

[**Expose MySQL running on localhost as service for minikube**][4]  
To access the local service (e.g. mysql) via `host.minikube.internal` which minikube exposes for [host access][1] on MacOS from k8s pods, a [service with a selector](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) is required.[^2] - You can grep the IP address needed for the Endpoint with: `minikube ssh 'grep host.minikube.internal /etc/hosts | cut -f1'`

Mapping a hostname to an IP

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mysql
spec:
  ports:
    - name: mysql
      protocol: TCP
      port: 3306
      targetPort: 3306
		  nodePort: 0

---
apiVersion: v1
kind: Endpoints
metadata:
	name: mysql
subsets:
- addresses:
	- ip: 192.168.64.1
	  ports:
		- port: 3306
		  name: mysql
```

After applying this configuration, you can then use `mysql` in the place of an IP address in any pod configuration.

If you want to expose an external service in the cloud to your pods we can map my hostname (CNAME).

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mysql-db-svc
namespace: external
spec:
	type: ExternalName
	externalName: mysql–instance1.123456789012.us-east-1.rds.amazonaws.com
```

## Deployment

### Staging

The job to deploy to the `vmw-test-cluster` runs when changes are merged into the `test` branch. You can gain access to the main cluster `vmw-test-cluster` using awscli with:

```
aws eks update-kubeconfig --name vmw-test-cluster
```

### Production

The job to deploy to the test cluster on runs when changes are merged into the `main` branch. You can gain access to the main cluster `vmw-capstone` using awscli with:

```
aws eks update-kubeconfig --name vmw-capstone
```

## Useful commands

Get the Kibana instance url

```
echo "https://$(kubectl get services kibana-kb-http | awk 'NR>1 {print $4}')/login"
```

Get the Elasticsearch/Kibana password (username: elastic)

```
echo $(kubectl get secret es1-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

Check/Watch Logstash pod logs for pipeline status

```
kubectl logs logstash --tail 1 --follow
```

## References

- [Migrating MySql Data Into Elasticsearch Using Logstash](https://qbox.io/blog/migrating-mysql-data-into-elasticsearch-using-logstash "Migrating MySql Data Into Elasticsearch Using Logstash")
- [Docker -> logstash input jdbc plugin for mysql database](https://github.com/dimMaryanto93/docker-logstash-input-jdbc)
- [Elasticsearch: Concepts, Deployment Options and Best Practices](https://cloud.netapp.com/blog/cvo-blg-elasticsearch-concepts-deployment-options-and-best-practices)
- [Elastic Cloud on Kubernetes | YouTube](https://youtu.be/WooqwWQv8UA?t=2024)
- [Add integration test for accessing "host.minikube.internal" in a container](https://github.com/kubernetes/minikube/issues/8439#issuecomment-799801736)

[1]: https://minikube.sigs.k8s.io/docs/handbook/host-access/
[2]: https://stackoverflow.com/questions/50952240/connect-to-local-database-from-inside-minikube-cluster
[3]: https://www.digitalocean.com/community/questions/how-to-allow-remote-mysql-database-connection
[4]: https://medium.com/@ManagedKube/kubernetes-access-external-services-e4fd643e5097
