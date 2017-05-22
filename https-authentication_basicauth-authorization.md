How to configure https and basicAuth for NCP &lt;--&gt; K8s Channel
===================================================================

**NOTE**: This document contains obsolete information and may be
deprecated in the future

1.  Git clone kubernetes. \[TODO(gangila): Shouldn't be needed decouple
    this\]

<!-- -->

    git clone https://github.com/kubernetes/kubernetes.git

2.  Generate certificates for your cluster.

<!-- -->

    cd kubernetes && cluster/saltbase/salt/generate-cert/make-ca-cert.sh <apiserver_ip>

3.  This will generate certificates in /srv/kubernetes:

\(a) ca.crt and ca.key which are the locally generated root certs, this
will be our local "Trusted" CA authority using which we sign other
certs.

\(b) kubecfg.crt and kubecfg.key are the certs which are used by kubelet,
controller-manager, scheduler.

(c) server.crt and server.key are used by the apiserver.

Now we can start the API server as:

    sudo /opt/bin/kube-apiserver --etcd_servers=http://127.0.0.1:4001
    --service-cluster-ip-range=10.31.1.0/24 --allow_privileged=false
    --kubelet_port=10250 —v=2 --client-ca-file=/srv/kubernetes/ca.crt
    --tls-cert-file=/srv/kubernetes/server.cert
    --tls-private-key-file=/srv/kubernetes/server.key
    --basic-auth-file=basicauth.csv

Here the basicauth.csv contains the username and password which NCP will
use to connect to the Kubernetes API server.

    nicira@kubenode:~/scripts$ cat basicauth.csv
    password,user name,user id
    admin,admin,1
    default,default,2

For the other components like kubelet, scheduler, and controller-manager
we define a kubeconfig like:

    nicira@kubenode:~/scripts$ cat kubelet.kubeconfig
    apiVersion: v1
    kind: Config
    clusters:
      - cluster:
          certificate-authority: /srv/kubernetes/ca.crt
          server: https://<apiserver_ip>:6443
        name: kubernetes
    contexts:
      - context:
          cluster: kubernetes
          user: kubelet
        name: kubelet-to-kubernetes
    current-context: kubelet-to-kubernetes
    users:
      - name: kubelet
        user:
          client-certificate: /srv/kubernetes/kubecfg.crt
          client-key: /srv/kubernetes/kubecfg.key

In each of these three components we pass a kubeconfig flag
--kubeconfig=kubelet.kubeconfig which configures them to communicate
them to the apiserver.

NOTE: Make sure that the certificates generated have read permissions
else the daemons won't be able to read them and fail to start.
