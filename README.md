# k8s-cluster-export

I wrote this quick and dirty bash script to export Kubernetes resources from a cluster in order to migrate to a new one. In my case we had to migrate multiple self-hosted Kubernetes cluster (including EBS PVCs) to AWS EKS.

Since on one hand the --export kubectl parameter is [depreciated](https://github.com/kubernetes/kubernetes/pull/73787) and on the other hand the goal from that server side export was more focused on a namespace migration on the same cluster (namespace removed while keeping for example clusterips) I decided to use the yaml export and remove unnecessary keys with yq.

### Supported resource types

Right now not all resources types are implemented.

```
available_resources=(
  pvc
  sts
  deploy
  svc
  cm
  ing
  secrets
)
```

## Usage

##### export all resource types for a namespace
```bash
./k8s-cluster-export -n test-ns
```

##### export all resources types and change the namespace
```bash
./k8s-cluster-export -n old-ns -c new-ns
```

##### export only PVCs and PVs
```bash
./k8s-cluster-export -n test-ns -r pvc
```

##### export only a limited list of PVCs and Deployments

This limitation is possible for all available resource types. The input file must match the resources type.

```bash
./k8s-cluster-export -n test-ns -r pvc -i pvc -r deploy -i deploy
```

deploy file
```
test-deploy-name1
test-deploy-name2
```

pvc file
```
pvc-name-1
pvc-name-2
```

## Migration procedure (on AWS with EBS PVCs)

*Disclosure: even if this is working in my  case, this has to be tested carfully. Since Kubernetes has the permission to delete you're EBS volumes such a migration should be planned with great caution.*

##### 1. export configs
```bash
./k8s-cluster-export -n test-ns
```
##### 2. scale down deployments / statefulsets on old Cluster
```bash
for i in */5_deploy/*; do
  deploy=$(yq r $i metadata.name)
  kubectl scale deployment --replicas 0 $deploy
done

for i in */6_sts/*; do
  sts=$(yq r $i metadata.name)
  kubectl scale statefulset --replicas 0 $deploy
done
```

##### 3. set tags for EBS volumes
```bash
OLD_CLUSTER=old-cluster-123
NEW_CLUSTER=new-cluster-123

for i in */2_pv/*; do
  vol=$(yq r $i spec.awsElasticBlockStore.volumeID | cut -f4 -d'/')
  aws ec2 delete-tags --resources $vol --tags Key=kubernetes.io/cluster/$OLD_CLUSTER
  aws ec2 delete-tags --resources $vol --tags Key=KubernetesCluster
  aws ec2 create-tags --resources $vol --tags Key=kubernetes.io/cluster/$NEW_CLUSTER,Value=owned
done
```

##### 4. crate snapshot from ebs volume
```bash
for i in */2_pv/*; do
  vol=$(yq r $i spec.awsElasticBlockStore.volumeID | cut -f4 -d'/')
  aws ec2 create-snapshot --volume-id $vol --description before-migration
done
```

##### 5. import config in new cluster
Take care to import PVCs first followed by PVs.
```bash
kubectl apply -f default/1_pvc/.
```

```bash
kubectl apply -f default/2_pv/.
```
check the pvc status and deploy all other resources after that.

```bash
kubectl apply -f default/3_cm/.
...
kubectl apply -f default/8_ing/.
```
