## Deploy php & nginx app using multi-node kube cluster


**Initialise kube control-panel node**
  
    kubeadm  init --pod-network-cidr=192.168.0.0/16

**Run the following commads to start using the cluster:**

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

**Setup Kube network policy using WeaveNet**

    sudo mkdir -p /var/lib/weave
    head -c 16 /dev/urandom | shasum -a 256 | cut -d" " -f1 | sudo tee /var/lib/weave/weave-passwd
    kubectl create secret -n kube-system generic weave-passwd --from-file=/var/lib/weave/weave-passwd
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

**Create a Kubernetes dashboard**

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
    kubectl apply -f config/dashboard/service_account.yaml
    kubectl apply -f confif/dashboard/cluster_role.yaml 
  
**Access the dashboard**

    #Open a local port and spin up a proxy server between local and kube server
    ssh -L 8001:127.0.0.1:8001 user@master-node-ip
    kubectl proxy

    #Acces the dashboard from the following link
    http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

    #Now, grab the access token from the service account we created earlier
    kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')


**Join the worker nodes to the master**

**Note**: if you have forgotten to note down the original join command from kubeadm init, run the below which will print the join command.


    kubeadm token create --print-join-command



#### **Setup php & nginx server using nfs for multi-node kube cluster**

  **Setup a nfs server on master node:**
 
  1) Install nfs server
        
        ```sudo apt install nfs-kernel-server ```
  2) create a nfs shared directory: 
  
        ```mkdir -p /srv/nfs/kubedata```
  3) Grant nfs share access
        ```
        vi /etc/exports
        
        #configure ip and access types
        /srv/nfs/kubedata 192.168.1.1/24(rw,no_root_squash)
        ```
  4) ```systemctl retsart nfs-server```
    
    
  **Create a persistent volume**
  
     
      kubectl apply -f nfs_pv.yml  
      kubectl apply -f nfs_pvc.yml
      
      #list the created pv's(should return a volume named nfs-pv with a claim nfs-pvc attached to it)
      kubectl get pv 
      
  
  
  **Setup nginx deployment & service:**
  
  1) create a nginx config map
  
     ```kubectl apply -f nginx_configMap.yaml ```
  2) Create a nginx deployment
  
      ```kubectl apply -f nginx_deployment.yaml ```
  3) Create a nginx service
  
        -----
      
        Sepcify one of your node's ip as the external ip for the nginx service.

            apiVersion: v1
            kind: Service
            metadata:
              name: nginx
              labels:
                tier: backend
            spec:
              selector:
                app: nginx
                tier: backend
              ports:
              - protocol: TCP
                port: 80
              externalIPs:
              - {external_ip}
              
      ```kubectl apply -f nginx_deployment.yaml ```

  
       
       

 **Setup php deployment & service:**
  
  1) Create a deployment
  
      ```kubectl apply -f php_deployment.yaml```
          
        This will deploy a php app with index.php file created uisng busybox image before the app is initiated.
  
            initContainers:
            - name: install
              image: busybox
              volumeMounts:
              - name: nfs-share
                mountPath: /var/www
              command: ['sh', '-c', 'echo "<?php echo phpinfo();" > /var/www/index.php']
      
  2) Create a service
  
      ```kubectl apply -f php_service.yaml```
               
  
  
Now the deployed app can by accessed by pointing your browser to **{external_ip}** used for nginx service at port 80.
http://{external_ip}


## WIN!
  
        
  
