# Kubernetes: Azure Service Operators

env:

docker-for-desktop

Azure Subscription

1.) Enable Kubernetes

2.) Follow the instructions here to deploy the latest version of Azure Service Operator on your Kubernetes cluster.

   Bare in mind, A.S.O is still in beta and not recommended for production environments.

   Steps are simple: a) deploy cert-manager, b) install Helm3 (if you dont have it already ;)), install ASO using Helm.

   <https://github.com/Azure/azure-service-operator>

3.) `kubectl get pods -n azureoperator-system`

   If all good, you must see something like this:

     NAME                                                READY   STATUS    RESTARTS   AGE
     azureoperator-controller-manager-5cfd5b7c4c-n9fsv   2/2     Running   0          2d15h

4.) `kubectl get crds`
     will show the available custom resource definitions.

In simple words, you have a k8s custom resource running in your local cluster.
This custom resource can access your Azure subscription and provision/manage azure resources defined in yaml.

5.) Create a resource group to verify that everything works as expected:

   `mkdir aso-test`

   `cd aso-test`

   `touch az-rg.yaml`

   Open the az-rg.yaml with your editor and add this:

   ```YAML
    apiVersion: azure.microsoft.com/v1alpha1
    kind: ResourceGroup
    metadata:
      name: az-k8s-operator
    spec:
      location: "northeurope"
   ```

6.) ~ /aso-test > `kubectl apply -f .`  

    Check your subscription, you must have a resource group: az-k8s-operator
    
    `> resourcegroup.azure.microsoft.com/az-k8s-operator created`

7.) Time to move faster. Inside your aso-test dir, create a yaml file for every definition and add each one of these:

   ```YAML
    apiVersion: azure.microsoft.com/v1alpha1
    kind: MySQLServer
    metadata:
      name: az-k8s-mysql-server # MySql Server
    spec:
      location: northeurope
      resourceGroup: az-k8s-operator
      serverVersion: "8.0"
      sslEnforcement: Disabled
      createMode: Default # Possible values include: Default, Replica, PointInTimeRestore (not implemented), GeoRestore (not implemented)
      sku:
        name: GP_Gen5_4 # tier + family + cores eg. - B_Gen4_1, GP_Gen5_4
        tier: GeneralPurpose # possible values - 'Basic', 'GeneralPurpose', 'MemoryOptimized'
        family: Gen5
        size: "51200"
        capacity: 4
   ```

   ```YAML
     apiVersion: azure.microsoft.com/v1alpha1
     kind: MySQLDatabase
     metadata:
       name: az-k8s-mysql-db  # MySql Server database
     spec:
       resourceGroup: az-k8s-operator
       server: az-k8s-mysql-server
   ```

   ```YAML
    apiVersion: azure.microsoft.com/v1alpha1
    kind: MySQLFirewallRule
    metadata:
      name: az-k8s-mysql-frwl-rl  # MySql Server Firewall Rule
    spec:
      resourceGroup: az-k8s-operator
      server: az-k8s-mysql-server
      startIpAddress: 0.0.0.0
      endIpAddress: 255.255.255.255
   ```

8.) `kubectl apply -f .`
     # Create the azure resources.

     > mysqldatabase.azure.microsoft.com/az-k8s-mysql-db created
       mysqlfirewallrule.azure.microsoft.com/az-k8s-mysql-frwl-rl created
       mysqlserver.azure.microsoft.com/az-k8s-mysql-server created
       resourcegroup.azure.microsoft.com/az-k8s-operator unchanged

9.) With the resources up and running, a Secret is created:

   `kubectl get secrets`

    > NAME                  TYPE                                  DATA   AGE
      az-k8s-mysql-server   Opaque                                4      4s

   `kubectl describe secret az-k8s-mysql-server`

    > Name:         az-k8s-mysql-server
      Namespace:    default
      Labels:       <none>
      Annotations:  <none>
      
      Type:  Opaque
      
      Data
      ====
      fullyQualifiedServerName:  44 bytes
      fullyQualifiedUsername:    30 bytes
      mySqlServerName:           19 bytes
      password:                  16 bytes
      username:                  10 bytes

   `kubectl delete -f .`
    Delete the Azure resources.

10.) The Secret is created at the end of the resources provisioning. We will use it in a Deployment,
     to create a WP intallation. The Deployment will wait for the resources to be created and then will
     deploy the WP Pod(s).

    ```YAML
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: wordpress
        namespace: default
        labels:
          app: wordpress
      spec:
        selector:
          matchLabels:
            app: wordpress
        template:
          metadata:
            labels:
              app: wordpress
          spec:
            containers:
              - name: wp-container
                image: microsoft/multicontainerwordpress
                envFrom:
                  - configMapRef:
                      name: wp-cm
                env:
                  - name: WORDPRESS_DB_HOST
                    valueFrom:
                      secretKeyRef:
                        name: az-k8s-mysql-server  # <-- Secret Name
                        key: fullyQualifiedServerName  # <-- Secret Key 
                  - name: WORDPRESS_DB_PASSWORD
                    valueFrom:
                      secretKeyRef:
                        name: az-k8s-mysql-server # <-- Secret Name
                        key: password # <-- Secret Key
                  - name: WORDPRESS_DB_USER
                    valueFrom:
                      secretKeyRef:
                        name: az-k8s-mysql-server # <-- Secret Name
                        key: fullyQualifiedUsername # <-- Secret Key
                resources:
                  limits:
                    memory: "128Mi"
                    cpu: "200m"
                ports:
                  - containerPort: 80
                volumeMounts:
                  - name: wp-persistent-storage
                    mountPath: /var/www/html
      
            volumes:
              - name: wp-persistent-storage
                persistentVolumeClaim:
                  claimName: wp-storage
    ```
11.) Here is the Service, the PVC and the ConfigMap for the Deployment.

   ```YAML
    apiVersion: v1
    kind: Service
    metadata:
      name: wp-service
      namespace: default
      labels:
        app: wordpress
    spec:
      selector:
        app: wordpress
      ports:
        - port: 80
          targetPort: 80
      type: LoadBalancer

   ```

   ```YAML
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: wp-storage
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: hostpath
      resources:
        requests:
          storage: 1Gi

   ```

   ```YAML
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: wp-cm
      namespace: default
    data:
      WORDPRESS_DB_NAME: az-k8s-mysql-db
      WORDPRESS_DEBUG: "1"
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_CACHE', true );
        define('WP_REDIS_HOST', 'redis-service');

   ```
12.) `kubectl delete -f .`
      To delete the existing resources. Give it a couple of minutes
      and verify the Azure resources have been deleted.

   `kubectl apply -f .`
    To create the Deployment with the Azure Resources.

      > mysqldatabase.azure.microsoft.com/az-k8s-mysql-db created
        mysqlfirewallrule.azure.microsoft.com/az-k8s-mysql-frwl-rl created
        mysqlserver.azure.microsoft.com/az-k8s-mysql-server created
        resourcegroup.azure.microsoft.com/az-k8s-operator created
        configmap/wp-cm created
        deployment.apps/wordpress created
        persistentvolumeclaim/wp-storage created
        service/wp-service created

   `kubectl get deployments --watch`

       > NAME        READY   UP-TO-DATE   AVAILABLE   AGE
         wordpress   0/1     0            0           0s
         wordpress   0/1     0            0           0s
         wordpress   0/1     0            0           0s
         wordpress   0/1     1            0           0s
         wordpress   1/1     1            1           2m57s

In Azure Portal, check the activity log in the Resource Group, to visualize what happens in the background.

Check the Service and click <http://localhost:80> to access the WP instalation page.

If, after Deployment is completed you see a database connection error, give it 30 seconds and try again.
