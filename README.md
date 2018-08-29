# Example-MutatingAdmissionWebhook

If pod definition yaml has a annotation "webhook.fujitsu.com/nginxserver",  
if it is, check if image name contains "nginx",  
if it is, change it to "nginx:1.14.0", and add a suffix "(mutated)" to pod name.  
Then set the nodeSelector of the pod. This makes sure the pod can be scheduled to a node that support webhook.

## Install as k8s service

[INSTALL_AS_SERVICE.md](https://github.com/charrywanganthony/Example-AdmissionWebhook/blob/master/INSTALL_AS_SERVICE.md)