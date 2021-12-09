# Working with repository


Check if there are any repositories 
```
helm repo list
Error: no repositories to show
```
# Add a repository 

`helm repo add bitnami https://charts.bitnami.com/bitnami`

```
helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```   

```
helm repo list
NAME    URL                               
bitnami https://charts.bitnami.com/bitnami
```

to search for mysql charts in repo 
```
helm search repo mysql
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/mysql           8.8.14          8.0.27          Chart to create a Highly available MySQL cluster  
bitnami/phpmyadmin      8.3.1           5.1.1           phpMyAdmin is an mysql administration frontend    
bitnami/mariadb         10.1.0          10.5.13         Fast, reliable, scalable, and easy to use open-...
bitnami/mariadb-cluster 1.0.2           10.2.14         DEPRECATED Chart to create a Highly available M...
bitnami/mariadb-galera  6.0.6           10.6.5          MariaDB Galera is a multi-master database clust...
```

to list for versions use --versions

`helm search repo mysql --versions`

to remove the repository 

`helm repo remove bitnami` 

# Now lets install a package

helm install mydb bitnami/mysql

```
helm install mydb bitnami/mysql
NAME: mydb
LAST DEPLOYED: Mon Dec  6 14:40:27 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 8.8.14
APP VERSION: 8.0.27

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace default

Services:

  echo Primary: mydb-mysql.default.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mydb-mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.27-debian-10-r35 --namespace default --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mydb-mysql.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"



To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'root.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace default mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
      helm upgrade --namespace default mydb bitnami/mysql --set auth.rootPassword=$ROOT_PASSWORD
```

you can retrieve the status again using `helm status mydb`

```
helm status mydb
NAME: mydb
LAST DEPLOYED: Mon Dec  6 14:40:27 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 8.8.14
APP VERSION: 8.0.27

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace default

Services:

  echo Primary: mydb-mysql.default.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mydb-mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.27-debian-10-r35 --namespace default --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mydb-mysql.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"



To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'root.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace default mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
      helm upgrade --namespace default mydb bitnami/mysql --set auth.rootPassword=$ROOT_PASSWORD
```

You cant create with same name 
```
helm install mydb bitnami/mysql
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```

you will have to create in different namesapce

`helm install mydb bitnami/mysql --namespace dev`

# List and uninstall 

to list the charts and info use `helm ls` or `helm list`

```
helm ls
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
mydb    default         1               2021-12-06 14:40:27.382952898 +0530 IST deployed        mysql-8.8.14    8.0.27     
```

also specific to a namespace 

`helm ls --namespace dev`

to uninsall chart we can use `helm uninstall mydb` or for a specific namespace `helm uninstall mydb --namespace dev`

```
helm uninstall mydb
release "mydb" uninstalled
```

# Helm Upgrade

```
ROOT_PASSWORD=$(kubectl get secret --namespace default mydb-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
      
      
helm upgrade --namespace default mydb bitnami/mysql --set auth.rootPassword=$ROOT_PASSWORD
```

you can also use values file `helm upgrade --namespace default mydb bitnami/mysql --values <path to values file>`

or you can use `--reuse-values` flag which will use the values that were taken during `installation`

`helm upgrade --namespace default mydb bitnami/mysql --reuse-values`

# How helm maintains release records 

whenever you do a release or a upgrade helm will create a secret for the release record . this secret will have entire information about that installation
when you will do a `helm uninstall` all information along with its secret will be gone

# Helm release work flow

```
1. Load the chart
2. Parse the values.yaml
3. Generate the yaml
4. Parse the yaml to kube objects and validate
5. Generate yaml and Send to kubernetes
```

# Helm --dry-run 

`helm install mydb bitnami/mysql --dry-run ` the resources will never get created to kubernetes cluster its a dry-run , way of seeing and testing what will happen

great way of debuuging our charts

# Helm template

`helm template  mydb1 bitnami/mysql`


it gives you all the manifests under the chart in kubernetes manifest format, it doesnot communicate with api server 

# More about release records 

for every release there will be secret created with release info base64 encoded

after upgrade again there will be another secret created related to that release

`helm upgrade mydb bitnami/mysql --values values.yaml`

```
kubectl get secrets
NAME                         TYPE                                  DATA   AGE
default-token-j87nj          kubernetes.io/service-account-token   3      6d3h
mydb-mysql                   Opaque                                2      115m
sh.helm.release.v1.mydb.v1   helm.sh/release.v1                    1      115m
sh.helm.release.v1.mydb.v2   helm.sh/release.v1                    1      108s
```

if you see for my db there are 2 secrets for 2 releases 

lets see whats inside the secret 

`kubectl get secret sh.helm.release.v1.mydb.v2`


# Helm get

to get release notes

`helm get notes mydb`

to get custom values

`helm get values mydb`

to get all default values

`helm get values mydb --all`

to get  values to a revision

`helm get values mydb --revision 2`

`helm get manifest mydb --revision 1` this will give you the manifest for revisions 

# Helm history 

`helm history` gives you the previous info


# Helm Rollback

rolling back to previous version 

`helm rollback mydb 1`

`helm rollback mydb 1`

```
helm history mydb
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
1               Mon Dec  6 19:21:40 2021        superseded      mysql-8.8.14    8.0.27          Install complete
2               Mon Dec  6 19:22:33 2021        superseded      mysql-8.8.14    8.0.27          Upgrade complete
3               Mon Dec  6 19:23:34 2021        deployed        mysql-8.8.14    8.0.27          Rollback to 1   
```

You can keep the history and still rollback, for that you need to provide `--keep-history` flag when you are uninstalling

`helm uninstall mydb --keep-history`

```
 helm history mydb
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION            
1               Mon Dec  6 19:28:48 2021        uninstalled     mysql-8.8.14    8.0.27          Uninstallation complete  
```

now we can rollback as well to the deleted helm chart of any version

```
helm rollback mydb 1

Rollback was success!

```















