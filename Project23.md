
### Data Persistence in Kubernetes

Have an EKS cluster set up. 

Containers are stateless by design, data does not persist in the containers. A container running in kubernetes pods will still be stateless unless its configured to support statefulness. 

To achieve statefulness in kubernetes, knowledge volumes, persistent volumes, and persistent volume claim will be applied.


### Volumes

On-disk files in a container are ephemeral, which presents some problems for non-trivial applications when running in containers. One problem is the loss of files when a container crashes. The kubelet restarts the container but with a clean state. A second problem occurs when sharing files between containers running together in a Pod. The Kubernetes volume abstraction solves both of these problems.

Docker has a concept of volumes, though it is somewhat looser and less managed. A Docker volume is a directory on disk or in another container. Docker provides volume drivers, but the functionality is somewhat limited.

Kubernetes supports many types of volumes. A Pod can use any number of volume types simultaneously. 
**Ephemeral volume** types have a lifetime of a pod, but **persistent volumes** exist beyond the lifetime of a pod. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts.

At its core, a volume is a directory, possibly with some data in it, which is accessible to the containers in a pod. How that directory comes to be, the medium that backs it, and the contents of it are all determined by the particular volume type used. This means, you must know some of the different types of volumes available in kubernetes before choosing what is ideal for your particular use case.

A list of a few of them - 

#### awsElasticBlockStore

An awsElasticBlockStore volume mounts an Amazon Web Services (AWS) EBS volume into your pod. The contents of an EBS volume are persisted and the volume is only unmmounted when the pod crashes, or terminates. This means that an EBS volume can be pre-populated with data, and that data can be shared between pods.
Creating an NGINX pod with data persistence using ```awsElasticBlockStore``` will have a config like below - 

```
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "<volume id>"
          fsType: ext4
EOF
```
**NOTE** - 
1. The volume section indicate the ttype of volume to be used to ensure persistence.

2. There is a  **volumeID** specified, therefore the **EBS** volume resource must be created prior the pod deployment, either in the AWS console or by using ```aws ec2 create-volume```

3. The nodes on which pods are running must be AWS EC2 instances
4. Those instances need to be in the same region and availability zone as the EBS volume
5. EBS only supports a single EC2 instance mounting a volume

So, create a pod without a volume. - So we can use its region and AZ to create the EBS volume then update the pod/deployment with the volume ID  of the EBS volume.

**For a Pod without volume**
``` 
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
```

Now that we have the pod running without a volume, Lets now create a volume from the AWS console.

1. In your AWS console, head over to the EC2 section and scroll down to the Elastic Block Storage menu.
2. Click on Volumes
3. At the top right, click on Create Volume 


Part of the requirements is to ensure that the volume exists in the same region and availability zone as the EC2 instance running the pod. 
For e.g - we have our EC2 instance on ```us-east-2b```.

Can the EBS volume -
 ``` 
  aws ec2 create-volume --availability-zone us-east-2b --size 10 --volume-type gp2
 ```
 Then upload the deployment yml  file with the volume ID of the EBS volume created.

 ![alt text](<images/Deployment for volume 1.jpg>)

 ![alt text](<images/4a describe po.jpg>)

Now this pod has a volume, it can persist data if the pod restarts. Although the pod can be used for stateful applications, the configuration is not yet complete. This is  because, the volume is not yet mounted to any specific filesystem inside the container. The directory  ``` /usr/share/nginx/html ``` whihc holds the software/website code is still ephemeral, and if there is any kind of update to the ```index.html``` file, the new changes will only be there for as long as the po is still running. If the pod dies after, all previously written data will be erased.

To complete the configuration, we will need to add another section to the deployment yaml manifest. The ```volumeMounts``` which basically answers the question "Where should this Volume be mounted inside the container?" Mounting a volume to a directory means that all data written to the directory will be stored on that volume.

**volumeMounts**

```
cat <<EOF | tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "	vol-07b537651bbe68be0"
          fsType: ext4
EOF
```
Notice the newly added section:

```
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/
``` 
* The value provided to name in volumeMounts must be the same value used in the volumes section. It basically means mount the volume with the name provided, to the provided mountpath