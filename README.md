# Jenkins with Kubernetes

This repository serves as an explanation for the various components of setting up Jenkins in a Kubernetes cluster, using minikube. Explanations are provided which may help others too.

## Quick Start

This is a simple command list to execute in order to get the cluster running, useful if you do not care about the explanation or for myself when starting things back up again in a clean environment.

    minikube start
    kubectl create namespace jenkins
    kubectl apply -f jenkins-pv.yml
    kubectl apply -f jenkins-serviceaccount.yml
    helm repo add jenkinsci https://charts.jenkins.io
    helm repo update
    helm install jenkins --namespace jenkins -f jenkins-values.yml jenkinsci/jenkins
    sleep 40 # Wait for the init containers to complete
    kubectl exec --namespace jenkins \
        -it svc/jenkins \
        -c jenkins \
        -- /bin/cat /run/secrets/chart-admin-password \
        && echo
    export NODE_PORT=$(kubectl get --namespace jenkins -o \
        jsonpath="{.spec.ports[0].nodePort}" services jenkins)
    export NODE_IP=$(kubectl get nodes --namespace jenkins -o \
        jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT/login

Take note of the initial Jenkins administrator password that is printed prior to logging in.

This is the installation process using Helm v3.

It's also possible to execute the following command, which allows for reaching Jenkins on `localhost:8080`

    kubectl port-forward -n jenkins jenkins-0 8080

## Minikube

* Install Minikube
* Run `minikube start` to create a single node kubernetes cluster.

## Kubernetes

Creating a separate namespace for Jenkins to be used in

    kubectl create namespace jenkins

From here, `Helm` is used in order to deploy Jenkins in a repeatable way.

### Persistent Volume

We create a persistent volume so that when our Kubernetes cluster (in this case, a single node with `minikube`) is shut down, we do not lose the data that Jenkins has saved, such as configuration etc. This is attached to the Jenkins controller pod, when using a production scale cluster with multiple nodes, we would instead mount a networked drive (NFS) to the cluster, such as AWS EFS or EBS.

    kubectl apply -f jenkins-pv.yml

### Service Account

Service accounts provide an identity to a pod. These are used by pods that wish to interact with the kube-apiserver. By default, all applications will authenticate as the default service account in the namespace they are in.

Here, a service account is created called `jenkins`

    kubectl apply -f jenkins-serviceaccount.yml

This leads into a `ClusterRole` which provides a set of permissions which can be assigned to a resource within a particular cluster. This works nicely as the Kubernetes API is organised in API groups, based on the API objects that they relate to (pod, deployment, service etc.); meaning that you can restrict the application to only be able to interact with specific API objects that are necessary for its operation.

A `ClusterRole` is used to define "cluster-wide" permissions. A `Role` is used to define the permissions within a namespace.

It is a `RoleBinding` that grants the particular permissions defined in a role to one or many users. It does this by holding a list of subjects (users, groups, or service accounts) with reference to the role that is being granted.

### Helm

[Helm](https://helm.sh/docs/intro/install/) is a package manager for Kubernetes, it bundles these into what is called a *chart*. The chart contains the information necessary to create an instance of an application on Kubernetes.

Once we have Helm, we can add the Jenkins repository to it.

    helm repo add jenkinsci https://charts.jenkins.io
    helm repo update

This adds the Jenkins repo under the name `jenkinsci` and then updates the local information of charts that are available.

We can then search for the charts within that repository using

    helm search repo jenkinsci

This searches for the keyword `jenkinsci` within the previously configured repository. This verifies the repository update.


#### Installing Jenkins

Jenkins is deployed using the Helm chart which includes the Kubernetes plugin. The values within `jenkins-values.yml` are defaults provided by the Jenkins team for the setup of the software.

We modify a few values in this file for the `minikube` setup.

    serviceType: NodePort # Changed from ClusterIP with Minikube, this isn't specified in the Jenkins docs.
    nodePort: 32000 # Changed from being commented out.
    storageClass: jenkins-pv # The PV we created earlier.
    serviceAccount:
        create: false # changed from 'true' in the defaults
        # Service account name is autogenerated by default
        name: jenkins # We already created the ServiceAccount earlier with this name.
        annotations: {}

Once we have done this, we can install it using `helm`.

    helm install <name> --namespace jenkins -f jenkins-values.yml <chart>

This equates to

    helm install jenkins --namespace jenkins -f jenkins-values.yml jenkinsci/jenkins

To get the Jenkins initial administrator password for setup, we can run

    kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/chart-admin-password && echo

To get the URL of Jenkins and the port, we can run

    export NODE_PORT=$(kubectl get --namespace jenkins -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
    export NODE_IP=$(kubectl get nodes --namespace jenkins -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT/login
