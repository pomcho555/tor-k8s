# tor-k8s

This project use minikube to start one locally cluster and deploy tor as service in k8s cluster

## How to use

* clone project

    git clone git@github.com:LeoCBS/tor-k8s.git


* Install minikube locally [(script copied from minikube project)](https://github.com/kubernetes/minikube#linux-continuous-integration-without-vm-support)

    chmod 777 minikube.sh
    ./minikube.sh

* Create tor deployment and Service

    kubectl create -f ./deploy.yaml
    kubectl create -f ./svc.yaml
