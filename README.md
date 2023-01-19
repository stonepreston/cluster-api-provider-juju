# cluster-api-provider-juju
// TODO(user): Add simple overview of use/purpose

## Description
// TODO(user): An in-depth paragraph about your project and overview of use

## Getting Started

### Creating the management cluster
You’ll need a Kubernetes cluster to run against. Assuming you have a juju controller, cloud, and cloud credentials set up already, you can add a new model and deploy Kubernetes-Core. Note: Your cluster will need storage capabilties, so ensure your overlay contains the cloud provider/integrator required to set that up. 
```sh
juju deploy kubernetes-core --overlay your-overlay.yaml --trust --channel 1.25/stable
```

After the deployment is active, copy the kubeconfig to your local machine:
```sh
juju scp kubernetes-control-plane/0:config ~/.kube/config
```

You can then create a storage class:
```sh
kubectl apply -f my-storageclass.yaml
```

For example, the YAML file for a vsphere storage class might look like this:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mystorage
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: zeroedthick
```

### Installing ClusterAPI
Install the ClusterAPI command-line tool clusterctl using [these instructions](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl).

Install cert-manager on your cluster:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

Run the init command to install ClusterAPI:
```sh
clusterctl init
```

Check the status to make sure things are running:
```sh
kubectl describe -n capi-system pod | grep -A 5 Conditions
```


### Running on the cluster
1. Clone this repo:
```sh
git clone https://github.com/charmed-kubernetes/cluster-api-provider-juju.git
cd cluster-api-provider-juju
```

2. If needed, build and push the manager image to the location specified by `IMG` (if `IMG` is left unset it will attempt to push the image into rocks using the `latest` tag, as this is the default value for the variable defined in the Makefile):
```sh
make docker-build docker-push IMG=<some-registry>/capi-juju-controller:tag
```

3. Install Custom Resources:
```sh
make install
```

	
4. Deploy the controller to the cluster with the image specified by `IMG` (if left unset it will use the default value for `IMG` defined in the Makefile):
```sh
make deploy IMG=<some-registry>/capi-juju-controller:tag
```

5. Verify the cluster and machine controllers have started by checking the controller-manager logs:
Find the name of the controller-manager pod:
```sh
kubectl get pods -n capi-juju-system
NAME                                            READY   STATUS    RESTARTS   AGE
capi-juju-controller-manager-78cc545778-8m6vl   2/2     Running   0          160m
```
Check the pod logs:
```sh
kubectl logs capi-juju-controller-manager-78cc545778-8m6vl -n capi-juju-system
```

You should see output similar to the following:
```sh
1.6733800930860527e+09	INFO	controller-runtime.metrics	Metrics server is starting to listen	{"addr": "127.0.0.1:8080"}
1.6733800930883508e+09	INFO	setup	starting manager
1.6733800930890276e+09	INFO	Starting server	{"path": "/metrics", "kind": "metrics", "addr": "127.0.0.1:8080"}
1.6733800930890732e+09	INFO	Starting server	{"kind": "health probe", "addr": "[::]:8081"}
1.6733800930981865e+09	INFO	Starting EventSource	{"controller": "jujucluster", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuCluster", "source": "kind source: *v1beta1.JujuCluster"}
1.6733800930982227e+09	INFO	Starting Controller	{"controller": "jujucluster", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuCluster"}
1.6733800930984538e+09	INFO	Starting EventSource	{"controller": "jujumachine", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuMachine", "source": "kind source: *v1beta1.JujuMachine"}
1.673380093098476e+09	INFO	Starting Controller	{"controller": "jujumachine", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuMachine"}
1.6733800930987048e+09	DEBUG	events	capi-juju-controller-manager-78cc545778-8m6vl_e14bf9b2-a388-43c3-9984-d441abef21b6 became leader	{"type": "Normal", "object": {"kind":"Lease","namespace":"capi-juju-system","name":"d18b2b62.cluster.x-k8s.io","uid":"52b2218f-a889-4d2e-b77a-60e20d03126a","apiVersion":"coordination.k8s.io/v1","resourceVersion":"21399"}, "reason": "LeaderElection"}
1.6733800931992176e+09	INFO	Starting workers	{"controller": "jujumachine", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuMachine", "worker count": 1}
1.6733800931992946e+09	INFO	Starting workers	{"controller": "jujucluster", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuCluster", "worker count": 1}
```

8. Create a secret containing your cloud credential data:
The cluster controller needs cloud credential information in order to create models and machines via juju. It expects the secret to be named with the format `<your-juju-cluster-name>-cloud-secret` and be created in the namespace you are going to create your cluster in. The command below will create a secret that should work with the sample resources that will be created in step 7.
```sh
kubectl create secret generic jujucluster-sample-cloud-secret --from-literal=username='your-username' --from-literal=password='your-password' --from-literal=vmfolder='your-vm-folder' -n default
```

7. Create the sample cluster resources:
```sh
kubectl apply -f config/samples/infrastructure_v1beta1_jujucluster.yaml -n default
```

After creating the sample resources, you can check the controller-manager logs again and see that it is doing work such as bootstrapping a Juju controller and creating other resource dependencies. 
Eventually it should create a model, at which point reconciliation should stop.
```sh
 manager 1.673380499749945e+09    INFO    Created model    {"response": {"ModelInfo":{"Name":"jujucluster-sample","UUID":"3e086dd2-2014-42a7-8685-1ba5dce6abc2","Type":"iaas","ControllerUUID":"ba7d6d4b-a892-4fb1-8500-cc904a48b75b", │
│ manager 1.6733804997502618e+09    INFO    Stopping reconciliation    {"controller": "jujucluster", "controllerGroup": "infrastructure.cluster.x-k8s.io", "controllerKind": "JujuCluster", "JujuCluster": {"name":"jujucluster-sample"
```

### Connecting the deployed controller to a local Juju client
1. Find the IP of the Juju controller service
```sh
kubectl get svc -n controller-jujucluster-sample-k8s-cloud
```

2. Create an ssh shuttle:
```sh
sshuttle -r ubuntu@<control-plane-IP> <svc-IP> --ssh-cmd 'ssh -i ~/.local/share/juju/ssh/juju_id_rsa'
```

Note that the first IP after the `@` is the IP of the Kubernetes-Control-Plane application as shown by running `juju status`. The second IP should be the IP of the controller service obtained above. 

3. Add the controller using the registration string:

The bootstrap job created by the cluster controller contains a registration string in its logs that you can use to add the controller to your local client. You can find it by first finding the name of the pod created by the boostrap job in the default namespace:
```sh
kubectl get pods -n default
```

Check the logs using:
```sh
kubectl logs <bootstrap-pod-name> -n default
```
There should be some lines present that look something like this:
```sh
Please send this command to capi-juju:
    juju register <registration-string>
```
Using your local juju client you can register the controller that was created. You will be promted to create a username and password:
```sh
juju register <registration-string>
```

4. Login to the admin account
The pod logs also contain the admin account information. Logout of the current account, and into the admin account, providing the password found in the pod logs:
```
juju logout
juju login -u admin -c jujucluster-sample-k8s-cloud
```

Once this is done, you should be able to list clouds, credentials, and models that exist on the controller, which is helpful for debugging. 

### Tearing things down
1.  Delete the sample resources:
```sh
kubectl delete -f config/samples/infrastructure_v1beta1_jujucluster.yaml -n default
```

After the resources are deleted, the controller will clean up after itself and remove the resource dependencies it created. You should see messages about resource removal like the one shown below in the controller-manager-logs:
```sh
 manager 1.6733908092290568e+09    INFO    Deleting Secret jujucluster-sample-juju-client  
...
```
 Part of this cleanup involves the removal of the juju controller namespace `controller-jujucluster-sample-k8s-cloud` and the various resources it contains, which can take several minutes. 
Before moving on, verify the Juju controller namespace was completely removed:
```sh
kubectl get ns
```
The namespace will be terminating for a while, but after several minutes should be removed the list.

2. Undeploy the controller
```sh
make undeploy
```

3. Uninstall the CRDs
```sh
make uninstall
```

## Contributing
// TODO(user): Add detailed information on how you would like others to contribute to this project

### How it works
This project aims to follow the Kubernetes [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)

It uses [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/) 
which provides a reconcile function responsible for synchronizing resources untile the desired state is reached on the cluster 

### Test It Out
1. Install the CRDs into the cluster:

```sh
make install
```

2. Run your controller (this will run in the foreground, so switch to a new terminal if you want to leave it running):

```sh
make run
```

**NOTE:** You can also run this in one step by running: `make install run`

### Modifying the API definitions
If you are editing the API definitions, generate the manifests such as CRs or CRDs using:

```sh
make manifests
```

**NOTE:** Run `make --help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2022.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

