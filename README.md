Instructions to deploy **Client IP Logger in MySQL DB** on GCP GKE Auto Pilot cluster
  1. Deploy GKE Auto Pilot cluster on GCP Console.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=10Gi \
        --set primary.persistence.storageClass=standard \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=10Gi \
        --set secondary.persistence.storageClass=standard \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Create a namespace **web** where the application will be deployed.
  5. Put the base64 encoded value of the **auth.rootPassword** set above in **mysql-secret.yaml** & then create the **Secret**.
     
     ```
     kubectl -n web apply -f mysql-secret.yaml
     ```
  6. Create the **ConfigMap** defined in **mysql-configmap.yaml**. The ConfigMap has the values for database host, name & user.
     
     ```
     kubectl -n web apply -f mysql-configmap.yaml
     ```
  7. Create the **Deployment** & **Service**. 

     ```
     kubectl -n web apply -f flask-deployment.yaml -f flask-service.yaml
     ```
  8. Deploy **Ingress**, **Managed Certificate** & **Frontend Config** which will create an ALB listening on port 443 by running below command. Before running below command, in the **Managed Certificate** put the domain name you need for your application.
     ```
     kubectl -n web apply -f ingress-all.yml
     ```
  9. Run ` kubectl -n yopass get ingress ` to retrieve the ALB IP ( Please wait couple of minutes ). Create an A record in **Cloud DNS** or your own DNS service pointing your domain name to this IP.
 10. Once the DNS entry has been added it will take couple of minutes ( can be 60 minutes in some cases ) for the certificate to be generated. Run ` kubectl -n yopass get managedcertificate ` or in GCP console to check for **Active** status.
 11. Access the app using `https://your_domain_name`.