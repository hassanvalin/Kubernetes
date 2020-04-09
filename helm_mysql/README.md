# Installed helm on the VM using below command and started 

$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh

$ helm version


# To create a chart directory along with the common files and directories used in a chart

helm create helm_first_app


# Once you make changes to values.yml and other files and you want to see how the deployment/service yml files looks like use below command that just replaces the values and prints for you , don't worry it will not change the actual files

helm template .


# Always better you use --help command

helm install --help


# To install the helm with any release use below command. If you want to install this in any of the namespace you can specify that as well

helm install <name_you_like> .


# To see the releases available use below command

helm list


# When you make changes you can do upgrade to the existing release using below command

helm upgrade <Name_you_used_earlier> .


# If you want to see the history on any release you can use below command. It gives Revision and other details as well

helm history <Name_you_used_earlier>


# When you get history if you want to rollback you can use rollback command with any available Revision versions

helm rollback <Name_You_Used_earlier> <Revision_Number_you_want_to_use>

# To delete the release 
helm delete <releasename>  //you can get the release name by running 'helm list'

# To see the metrics on the pods which is under particular namespace
kubectl top pods -n <Namespace>


# If watch is installed on your VM you can use below command to see the dynamic change of CPU and memory
watch kubectl top pods -n <your_Name_Space>

# If you want to watch during pods recreation use below command
watch kubectl get pods -n <your_Name_Space>

# In any case if you want to edit the running deployment use below command that will recreate the Pod as well
kubectl edit deployment <Your_deployment_Name> -n <your_Name_Space>



#  ########### Using Volumes ############################

- templates/pv.yaml

Persistent volume has been created

- templates/pvc.yaml

Persistent volume claim has been created

- templates/mysql-deploy-svc.yaml

Mysql DB has been created and exposed the port with service


-  If you want to apply through kubectl you can generate yaml files using 'helm template .' that gives you all yaml files, naming it as test2.yaml

---
# Source: first_app/templates/pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
        name: my-first-app-pv
        namespace: helm-ns1
        labels:
            app: persistent
            release: v1
spec:
        accessModes:
                - ReadWriteMany
        hostPath:
                path: /opt/mysql-data
                type: DirectoryOrCreate
        capacity:
                storage: 1Gi
---
# Source: first_app/templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
        name: my-first-app-pvc
        namespace: helm-ns1
        labels:
            app: persistent
            release: v1
spec:
        accessModes:
                - ReadWriteMany
        resources:
                requests:
                     storage: 1Gi
        selector:
                matchLabels:
                    app: persistent
                    release: v1
---
# Source: first_app/templates/mysql-deploy-svc.yaml
apiVersion: v1
kind: Service
metadata:
   name: mysql-deployment
   namespace: helm-ns1
   labels:
      app: mysql
spec:
   ports:
      - name: http
        port: 3306
   selector:
        app: mysql
---
# Source: first_app/templates/mysql-deploy-svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: mysql-deployment
    namespace: helm-ns1
    labels:
       app: mysql
spec:
    selector:
       matchLabels:
            app: mysql
    template:
       metadata:
            labels:
               app: mysql
       spec:
            containers:
               - name: mysqlapp
                 image: mysql:5.7
                 env:
                    - name: MYSQL_ROOT_PASSWORD
                      value: secretpwd
                 volumeMounts:
                    - name: mysql-volume
                      mountPath: /var/lib/mysql
            volumes:
              - name: mysql-volume
                persistentVolumeClaim:
                      claimName: my-first-app-pvc



- Created a namespace helm-ns1 and applied the file using 'kubectl apply -f test2.yaml' that creates all below kubernetes objects

persistentvolume/my-first-app-pv created
persistentvolumeclaim/my-first-app-pvc created
service/mysql-deployment created
deployment.apps/mysql-deployment created


- You can see pv and pvc created using command 'kubectl get pv'

- Check deployments, pods, svc using below kubectl commands

kubectl get deployments -n helm-ns1
kubectl get pods -n helm-ns1
kubectl get svc -n helm-ns1


- If you want to verify by connecting to the mysql DB that created use below sample command that connects to your DB

kubectl run -it --rm --image=mysql:5.7 --restart=Never -n helm-ns1 mysql-client -- mysql -h <your_svc_name> -psecretpwd


If you don't see a command prompt, try pressing enter.

mysql> create database mydb;
Query OK, 1 row affected (0.00 sec)

mysql> use mydb;
Database changed

mysql> create table mytable(name varchar(20), age varchar(15));
Query OK, 0 rows affected (0.01 sec)

mysql> select * from mytable;
Empty set (0.01 sec)


- Even if you delete the pod and recreate a new one that data will be available in DB because of volumes

