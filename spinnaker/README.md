# Deploy Spinnaker in K8S with hal

1. We need to have Minikube and Minio running on VM where ever you are performing operations

- Minikube

- minio.yaml : Create a container in cluster ( kubectl apply -f minio.yaml )   // We are using Minio as Storage service


---
2. Install spinnaker using Minikube and Minio. Run below command that creates a docker container and make sure to have config files and certs available in folders in server VM before mounting onto container

docker run\
    --name halyard -d \
    -v ${PWD}/hal:/home/spinnaker/.hal \
    -v ${PWD}/kube:/home/spinnaker/.kube \
    -e KUBECONFIG=/home/spinnaker/.kube/config \
    gcr.io/spinnaker-marketplace/halyard:1.28.0


- Create a folder spinnaker inside /opt to keep all our work related to this and create folders 'hal' and 'kube' that gets mounted onto container

- Get the kube config file, certs into kube folder on server VM and modify the cert paths inside config file. The place where these certs would be going on the container is '/home/spinnaker/.kube'


- The above command will create a docker container with name 'halyard'. You can login to the container with 'docker exec -it halyard bash' and see the folders and created there

- You might get permission errors on the docker container when running these files/certs , this is because of different file permissions compared to the server VM and container. To avoid this issue please set the user id to '1000' for these files on server VM with command 'chown 1000 *' , where 1000 will set the actual username on the container for these files

- You may not edit files on running docker container in that case you can edit files on the server VM because we have created mount points from server VM to docker container
 

---
3. Create a spinnaker service account in kubernetes cluster

- CONTEXT=$(kubectl config current-context)           // It gives the current context from the config file

- kubectl apply --context $CONTEXT -f https://spinnaker.io/downloads/kubernetes/service-account.yml

// Use below commands to verify the created service account and clusterrolebinding

- kubectl get sa -n spinnaker

- kubectl get clusterrolebinding -n spinnaker


TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)


- kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

- kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user


---
4. hal config file 'config' has to be present inside .hal folder on docker container with our process in previous steps if this file is not created just run the command 'hal config' to get the file created. If you run this on your server VM hal folder as well it gets reflected on the docker container because of mount path

We need to make some changes to the config file to support our deployments ...

- Enable Kubernetes incase of any K8S Deployment ( You can see different cloud providers in /opt/spinnaker/hal/config file, in our case we are going to use Kubernetes ) by running below command

hal config provider kubernetes enable

CONTEXT=$(kubectl config current-context)      //Getting the current-context value

// We need to add an account under K8S provider section with some name and context we are getting above and also make sure you get the right kube config file pointed in this section as well

hal config provider kubernetes account add minikube_k8s \
    --provider-version v2 \
    --context $CONTEXT

// Set the artfacts to true under features section with below command

hal config features edit --artifacts true


---
5. We are using Minio as storage service by deploying minio.yaml and exposing service as NodePort. Given that minio doesn't support versioning objects , we have to disable it in Spinnaker. Create the file '/.hal/default/profiles/front50-local.yml' and add below line to it

- spinnaker.s3.versioning: false


// Default MINIO_SECRET_KEY and MINIO_ACCESS_KEY are minioadmin/minioadmin , The endpoint is the IP and NodePort which got exposed for Minio service

echo $MINIO_SECRET_KEY | hal config storage s3 edit --endpoint $ENDPOINT \
    --access-key-id $MINIO_ACCESS_KEY \
    --secret-access-key

// Ex: echo minioadmin | hal config storage s3 edit --endpoint http://<IP>:<NodePort> --access-key-id minioadmin --secret-access-key

- hal config storage edit --type s3

// With the above command it updates the s3 section with storage details in hal config file


---
6. Set the deploymentEnvironment details with below commands

- hal config deploy edit --type distributed --account-name minikube_k8s           //minikube_k8s is the account that was created earlier

- hal version list         //It gives list of hal versions

- hal config version edit --version $VERSION            //Take the version from the list and set it here

- hal deploy apply             // It will deploy the spinnaker in Kubernetes


---
7. Updated the security api base url in hal config file with below command ...

- hal config security api edit --override-base-url http://<IP>:<NodePort> 

//With the hal deploy it creates some services and default ports but those ports are not working on my server VM so I've done port forwarding as NodePort with supported port numbers and that is configured above


---
8. When you open http://<IP>:<Gate_Port> for Spinnaker homepage , you might see Cores blocking issue , so I've updated config file

security:
    apiSecurity:
      ssl:
        enabled: false
      overrideBaseUrl: http://<IP>:<NodePort_Of_API>
      corsAccessPattern: http://<IP>:<NodePort_Of_Deck>


---
9. Below are the basic commands to verify the deployments/pods/services

- kubectl get deployments -n spinnaker

- kubectl get pods -n spinnaker

- kubectl get svc -n spinnaker

// If you want to expose any deployment as service use below command 

- kubectl expose deployment/<Deployment_Name> --type=NodePort --name <Name_You_Like> -n spinnaker

//If you want to edit any running service/deployment inside kubernetes

- kubectl edit svc <Name_You_Like> -n spinnaker


