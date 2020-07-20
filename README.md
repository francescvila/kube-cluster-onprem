# Table of Contents

* [Introduction](#introduction)
* [Goals](#goals)
* [Prerequisites](#prerequisites)
* [Workspace](#workspace)
* [SSH Remote Servers](#ssh-remote-servers)
* [Kubernetes Dependencies](#kubernetes-dependencies)
* [Master Node](#master-node)
* [Worker Nodes](#worker-nodes)
* [Verifying the Cluster](#verifying-the-cluster)
* [Running an Application](#running-an-application)
* [Ingress Controller](#ingress-controller)
* [TLS Certificate](#tls-certificate)
* [Clean up](#clean-up)
* [Conclusion](#conclusion)
* [References](#references)

## Introduction

DevOps culture requires gradual change management.
To provide stability in the content of a high rate of code change we need robust tools for managing our infrastructure.
Orchestration uses automation to perform workflows and processes to manager our applications and infrastructure.
Kubernetes is a powerful container orchestration system that manages containers at scale.
Version 1.14 of Kubernetes will be used, as it's the official supported version at the time of this post's publication.

Kubeadm will be used for automating the installation and configuration of Kubernetes components.
We will use a configuration management tool like Ansible for installing operating-system-level dependencies.
Using Ansible makes creating additional clusters or recreating existing clusters much simplier.

## Goals

Our Kubernetes cluster will include one master node and two worker nodes.
We will need three machines to be provided. They can be either VMs, cloud instances, or physical machines.
In case of having created VMs on a VMware server we will have to ensure that hardware virtualization setting is enabled.
The master node is the server responsible for managing the state of the cluster.
The worker nodes are the servers where our services will run.
A cluster capacity can be increased by adding more worker nodes.

## Prerequisites

* We will need an SSH key pair on our local Linux machine.
* Our three servers are running Debian 10 (buster) with at least 2GB RAM and 2 vCPUs each.
* We are able to SSH into each server as the root user with our SSH key pair.
* Ansible is installed on our local machine. We are familiar executing Ansible playbooks.

> Considering the original DO's tutorial post required Debian 9 for execution (see #references), some fixes have been made to make it run on Debian 10 and Ansible version 2.9.9.

## Workspace

You might clone this project on our local machine that will serve as our workspace. Then you `cd` into the project folder.
We will configure Ansible so that it can communicate with our remote servers and execute commands on them.
We will create a `hosts` file containing inventory information of servers and groups.
You might use the provided `hosts` file as an example.

The master node IP is displayed as `master_ip`.
The worker nodes IP is displayed as `worker_1_ip` and `worker_2_ip`.

In the `hosts` file we specify the logical structure of our cluster.

```yaml
# hosts

[masters]
master ansible_host=master_ip ansible_user=root

[workers]
worker1 ansible_host=worker_1_ip ansible_user=root
worker2 ansible_host=worker_2_ip ansible_user=root

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

> Ansible version 2.9 requires the use of Python version 3+ interpreter on remote server.

## SSH Remote Servers

Considering that in `hosts` file servers are going to be accessed with ansible_user root, and we cannot ssh to any Debian server as root, we will have to manually add our public id_rsa key to each cluster machine.

```
cat ~/.ssh/id_rsa.pub
ssh username@server_ip
su -
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "<public-ssh-key-string>" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
exit
exit
```

We can check Ansible is able to connect to all servers by gathering the ansible_hostname.

```
ansible all -i hosts -m setup -a 'filter=ansible_hostname'
``` 

We will create a non-root user with sudo privileges on all servers so that we can SSH into them manually as an unprivileged user.

In the file named `init.yml` we have a *play* of Ansible steps to create our non-root user with sudo privileges.

Execute this playbook on your local machine:

```
ansible-playbook -i hosts ./init.yml
```

## Kubernetes Dependencies

We will install the operating-system-level packages required by Kubernetes with Debian’s package manager (APT).
The packages required are: `docker`, `kubeadm`, `kubelet` and `kubectl`.
Swap will also be disabled since kubernetes cannot work with swap enabled.

With the playbook placed in file `kube-dependencies.yml` we install all Kubernetes dependencies in each node as required.

> `Kubectl` is only required to be installed in our master node. and

Execute this playbook on your local machine:

```
ansible-playbook -i hosts ./kube-dependencies.yml
```

## Master Node

We will set up the master node.
It is important to check if port forwarding is enabled.
Without port forwarding enabled pods will not be accessible.

Execute the playbook in `master.yml` on your local machine:

```
ansible-playbook -i hosts ./master.yml
```

The cluster is initialized on the master node.
The private subnet that the pod IPs will be assigned from is specificied.
The Kubernetes API server is exposed using master node public IP.
Flannel is used above the subnet to connect all the cluster nodes.
Kubernetes port is checked so Ansible playbook does not crash if the cluster has been already set up.
Kubernetes config file is copied to operator's `~/.kube` folder so that the cluster can be managed more securely by the operator user.

## Worker Nodes

Adding nodes to the cluster involves having the necessary cluster information, such as the IP address and port of the master’s API Server, and a secure token.

Only nodes that pass in the secure token will be able join the cluster.

We get the join command from master node and join all worker nodes to the cluster.

It is important to check if port forwarding is enabled.

Execute the playbook in `workers.yml` on your local machine:

```
ansible-playbook -i hosts ./workers.yml
```

## Verifying the Cluster

In order to verify if all cluster nodes are up and running and connectivity between the master node and workers is working correctly, we can log in the master node (as operator user) and ensure that all nodes are ready.

```
kubectl get nodes
```

If all of your nodes have the value Ready for STATUS, it means that they are part of the cluster and ready to run workloads.

## Running and Application

We can now deploy any containerized application to our cluster.
We can deploy Nginx using deployments and services to see how this application can be deployed to the cluster.

Within the master node, we execute the following command to create a deployment named nginx:

```
kubectl create deployment nginx --image=nginx
```

This deployment will create a pod with one container from the Docker registry’s Nginx Docker image.

We create a service named nginx that will expose the pod publicly. It will do so through a NodePort, a scheme that will make the pod accessible through an arbitrary port opened on each node of the cluster. This way, we will be able to access the pod from outside the cluster.

```
kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
```

Run the following command to check the status of the recently created service:

```
kubectl get services
```

To test that everything is working, visit `http://worker_1_ip:nginx_port` or `http://worker_2_ip:nginx_port` through a browser on your local machine. We can either visit `http://master_ip:nginx_port`. It works, as well.

We have to bear in mind that `nginx_port` is a port number assigned locally by a cluster component (kube-proxy) to the service object that exposes the Nginx pod. It is not the nginx port exposed within the container's pod itself.

You can get the assigned node port number using this command:

```
kubectl get -o jsonpath="{.spec.ports[0].nodePort}" services nginx && echo
```

Change master and workers IPs with your own server IPs.

To remove the Nginx application, first we delete the nginx service from the master node:

```
kubectl delete service nginx
```

Then we delete the deployment:

```
kubectl delete deployment nginx
```

We run these commands to check if it worked correctly:

```
kubectl get services
kubectl get deployments
```

## Ingress Controller

Accessing services externally through a node IP and a random port might be good for test cases.
But if we want to access our website using a domain name and secure our connection with HTTPS we need to use a Kubernetes controller called Ingress.
Ingress will redirect external requests to our internal Kubernetes service.

We will set up a pod, a service, an ingress object and a namespace for a simple nginx application.

We apply the `nginx-app.yaml` file:

```
kubectl apply -f nginx-app.yml
```

Creating an ingress component is not enough. We need an implementation for Ingress, which is called Ingress Controller.
An ingress controller evaluates all the rules and manages all redirections.
Ingress controller will be our entrypoint to the cluster.

For installing and Ingress Controller we can use the official deploy repository for Kubernetes' bare-metal installation.
The best way to install it, is using Helm within the master node.

Execute the playbook in `ingress.yml` on your local machine.

```
ansible-playbook -i hosts ./ingress.yml
```

Now we can access our demo Nginx application by faking the domain name to the ingress controller host (`master_ip`).
Change `master_ip` with yours.

```
curl --header "Host: nginxapp.com" http://`master_ip`
```

We can also check the default backend that comes with ingress controller, in case your request does not match a configured domain name in your ingress rules (for example, `nginxapp.com`).

```
curl --header "Host: nginxapp.com" http://`master_ip`
```

For the sake of sanity we can delete all the components created for the `nginx-app` example by executing:

```
kubectl delete -f ./nginx-app.yml
```

## TLS Certificate

In most occasions, we want to enable a TLS entrypoint to our website, which is hosted in the cluster.

To configure HTTPS forwarding in ingress we need to define the attribute TLS above the rules section, with a host and a secret name, which is a reference of a secret that we have to create in our cluster. This secret will hold our TLS secret certificate and key.

To create the SSH public key and public certificate pair we execute:

```
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -nodes -days 365 -subj '/CN=master_ip'
```

Change `master_ip` with yours.

To encode the key and certificate pair in base64 encryption we execute:

```
cat tls.crt | base64 | tr -d '\n' && echo
cat tls.key | base64 | tr -d '\n' && echo
```

We copy and paste the values on each key in the data section of the secret component.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-app-secret-tls
  namespace: nginx-app-ns
data:
  tls.crt: <paste here base64 encoded cert>
  tls.key: <paste here base64 encoded key>
type: kubernetes.io/tls
```

We apply the `nginx-app-tls.yaml` file:

```
kubectl apply -f ./nginx-app-tls.yml
```

Now we can access our demo Nginx application by faking the domain name to the ingress controller host.
In the `curl` command we have to specify the insecure option, as our certificate is self-signed.

```
curl --insecure --header "Host: nginxapp.com" https://`master_ip`
```

Change `master_ip` with yours.

We can delete all the components created for the `nginx-app-tls` example by executing:

```
kubectl delete -f ./nginx-app-tls.yml
```

## Clean up

In case you need to clean up any Kubernetes node or all of them you just have to execute the next command in every node you want to clean up:

```
kubeadm reset
```

This command performs the best effort to revert all changes made by `kubeadm init` or `kubeadm join`.

## Conclusion

Systems administrators cannot always create clusters on public cloud platforms for development purposes. They are "encouraged" to reuse physical machines which were decommissioned in the past, after having migrated from on-premise to cloud infrastructure.

The main objective of sharing this to tutorial is to provide a concise and effective guide to set up a Kubernetes cluster whenever is needed on premise to quickly create some development environment.

This tutorial is based on an excellent post published by [bsder](https://www.digitalocean.com/community/users/bsder) on Digital Ocean.
Several corrections and additions have been made as new requierements have been introduced.

Check out [The Kubernetes Official Documentation](https://kubernetes.io/docs/) to learn more about concepts and features that Kubernetes has to offer.

The author selected a free and open source license to share its post.
Feel free to fork this project and update it for your own needs.

I hope it helps.

## References

* [How To Create a Kubernetes Cluster Using Kubeadm on Debian 9](https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-debian-9)
* [Disabling Swap for Kubernetes in an Ansible Playbook](https://germaniumhq.com/2019/02/14/2019-02-14-Disabling-Swap-for-Kubernetes-in-an-Ansible-Playbook/)
* [NGINX Ingress Controller Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/)
