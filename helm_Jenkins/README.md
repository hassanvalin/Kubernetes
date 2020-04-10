# When you apply deployment.yaml the pod will not get success and it throws an error message where it unable to touch a log file under the mount we created /var/jenkins_home

To avoid that issue we have to change the folder permissions in your server before creating the containers with deployment.yaml file

Go to /opt location in your actual server where you are initiating the 'kubectl apply -f deployment.yaml' and change the user to 1000 with command 'chown -R 1000 jenkins_home' then it will be as below

pwd ---- /opt

ls -lrt 

drwxr-xr-x. 13      1000 root 4096 Apr 10 00:24 jenkins_home

Now try to run the command 'kubectl apply -f deployment.yaml' , then it will work successfully
