# Ubuntu 22.04 Home Lab Build

## Considerations

* The following was built on ProxMox 7.2-7.  The environment is highly available.

* Hosts were pinned to specific IPs in the firewall.  

  * Current environment is built on top of a UniFi network.

* Pi-Hole is the DNS server, hosts were defined there to ease burden of communication.

## Build Process

1. Set up hosts accordingly with your network.
2. Update inventory.
3. Execute kube_base.yaml for base buildup on servers.

    `ansible-playbook -i inventory kube_base.yaml -K`

4. Sometimes issues are present with hardware clock, fix this by setting the correct timezone.

    `sudo timedatectl set-timezone America/New_York`

5. Consider Setting Up Load Balancer

    * Due to the HA nature of Kubernetes, the master nodes should be set up in an HA like fashion.

    * Set up nginx load balancer on Fedora LXC containter:

      * `dnf update`

      * `dnf install nginx`

      * `dnf install nginx-mod-stream`

      * Modify `/etc/nginx/nginx.conf` to include the following:

          ```txt
          user nginx;
          worker_processes 4;
          error_log /var/log/nginx/error.log;
          pid /run/nginx.pid;
          worker_rlimit_nofile 40000;

          # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
          include /usr/share/nginx/modules/*.conf;

          events {
              worker_connections 8192;
          }

          stream {
              upstream servers_https {
                  least_conn;
                  server 10.0.3.231:6443 max_fails=3 fail_timeout=5s;
                  server 10.0.3.232:6443 max_fails=3 fail_timeout=5s;
              }
              server {
                  listen 6443;
                  proxy_pass servers_https;
              }
          }
          ```

      * Restart the service.

6. Start the install on the first primary master:

    `sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --service-cidr=10.18.0.0/24 --upload-certs --control-plane-endpoint=kubenginxlb.ward.lab`

7. Wait for the first primary master to become ready.  You should see a message `Your Kubernetes control-plane has initialized successfully!`.  Follow the prompts to set up additional masters and workers.

8. Join the nodes:

    * Review the output generated to connect additional workers. 

    * If additional masters need to be joined, the following should be done:

        * Ensure keys are uploaded from primary master:

          `sudo kubeadm init phase upload-certs --upload-certs`

          Ensure you get the certificate key (part of output from command). Example:

          ```[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
          [upload-certs] Using certificate key:
          92081447c52a73d2297a704f7b0554a4b9c181968787f3b041f18615a8190e13
          ```

        * Print the join command

          `kubeadm token create --print-join-command`

        * Add the following flags to the above join command

          `--control-plane --certificate-key "upload certs section"`

        * Command should look like below

          `sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key 92081447c52a73d2297a704f7b0554a4b9c181968787f3b041f18615a8190e13`

    * From the primary master node, pull down the requisite kube information for authentication to the backend control plane.,

        ```bash
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
        ```

9. Configure Network Plugin:

    * Apply configuration:

      `kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml`

    * Nodes should now be able to see one another.

    * Validate that all pods are in a *Running* state.

        ```bash
        NAMESPACE      NAME                                   READY   STATUS    RESTARTS      AGE
        kube-flannel   kube-flannel-ds-7qgvf                  1/1     Running   0             6m7s
        kube-flannel   kube-flannel-ds-cccv2                  1/1     Running   0             6m7s
        kube-flannel   kube-flannel-ds-kmjsq                  1/1     Running   0             6m7s
        kube-flannel   kube-flannel-ds-sdsp6                  1/1     Running   0             6m7s
        kube-system    coredns-565d847f94-8mftt               1/1     Running   0             28m
        kube-system    coredns-565d847f94-wn9qg               1/1     Running   0             28m
        kube-system    etcd-kubemaster01                      1/1     Running   0             28m
        kube-system    etcd-kubemaster02                      1/1     Running   0             26m
        kube-system    kube-apiserver-kubemaster01            1/1     Running   0             28m
        kube-system    kube-apiserver-kubemaster02            1/1     Running   0             26m
        kube-system    kube-controller-manager-kubemaster01   1/1     Running   1 (26m ago)   28m
        kube-system    kube-controller-manager-kubemaster02   1/1     Running   0             26m
        kube-system    kube-proxy-58tkc                       1/1     Running   0             26m
        kube-system    kube-proxy-hv5fd                       1/1     Running   0             15m
        kube-system    kube-proxy-jfbd5                       1/1     Running   0             28m
        kube-system    kube-proxy-tlz9x                       1/1     Running   0             16m
        kube-system    kube-scheduler-kubemaster01            1/1     Running   1 (26m ago)   28m
        kube-system    kube-scheduler-kubemaster02            1/1     Running   0             26m
        ```

10. Configure External Access

    * From the master, copy the context from the master server to the local machine.

    * Guide: <https://sarasagunawardhana.medium.com/kubernetes-configure-context-and-switching-contexts-in-multiple-clusters-part-4-e93677737328>

11. Install MetalLB

    1. Follow the guide outlined here.

        * Ensure that `strictARP: true`.

          ```bash
          # see what changes would be made, returns nonzero returncode if different
          kubectl get configmap kube-proxy -n kube-system -o yaml | \
          sed -e "s/strictARP: false/strictARP: true/" | \
          kubectl diff -f - -n kube-system

          # actually apply the changes, returns nonzero returncode on errors only
          kubectl get configmap kube-proxy -n kube-system -o yaml | \
          sed -e "s/strictARP: false/strictARP: true/" | \
          kubectl apply -f - -n kube-system
          ```

        * Follow the Installation By Manifest: <https://metallb.universe.tf/installation/#installation-by-manifest>

          ```bash
          kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.5/config/manifests/metallb-native.yaml
          ```

    2. Configure:

        * Define IPs to use in an `IPAddressPool`, try to reserve space that is unused:

          ```yaml
          apiVersion: metallb.io/v1beta1
          kind: IPAddressPool
          metadata:
            namespace: metallb-system
            name: first-pool
          spec:
            addresses:
            - 10.0.2.200-10.0.2.215
          ```

        * Set up the correct IP Announcement for Layer 2 Configuration:

          ```yaml
          apiVersion: metallb.io/v1beta1
          kind: L2Advertisement
          metadata:
            name: example
            namespace: metallb-system
          ```

    3. Applications should now be able to utilize the `IPAddressPool`:

        * Set up example application:

          ```bash
          kubectl create deployment nginx-app --image=nginx --replicas=2
          ```

        * Exposing a services as type Load Balancer should correctly assign an address from the pool.

          ```bash
          kubectl expose deploy/nginx-app --type=LoadBalancer --port=80 
          ```

        * Notice the Exteral-IP:

          ```bash
          toddwardzinski@kubemaster01:~$ kubectl get svc
          NAME         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
          kubernetes   ClusterIP      10.18.0.1     <none>        443/TCP        5h2m
          nginx        LoadBalancer   10.18.0.235   10.0.2.200    80:32676/TCP   97m

          ```
## Issues Run Into &amp; Resolves

* Removal of flannel CNI when things go bad.

  * On all nodes:

      ```text
      1. Find the previous flannel yaml file and execute:

      kubectl delete -f xxxx.yaml

      2. Delete the cni configuration file

      rm -rf/etc/cni/net.d/*

      3. Restart kubelet, if not, restart the server reboot

      systemctl restart kubelet

      Then check kubectl get pod -n kube-system
      ```

## References

* <https://metallb.universe.tf/configuration/>
