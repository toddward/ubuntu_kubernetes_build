# Ubuntu 22.04 Home Lab Build

## Build Process

1. Set up hosts accordingly with your network.
2. Update inventory.
3. Execute kube_base.yaml for base buildup on servers.

    `ansible-playbook -i inventory kube_base.yaml -K`

4. Sometimes issues are present with hardware clock, fix this by setting the correct timezone.

    `sudo timedatectl set-timezone America/New_York`

5. Start the install on the first primary master:

    `sudo kubeadm init --control-plane-endpoint=k8smaster.example.net`

6. Wait for the first primary master to become ready.  You should see a message `Your Kubernetes control-plane has initialized successfully!`.  Follow the prompts to set up additional masters and workers.

7. Join the nodes:

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

8. Configure Network Plugin:

    * Validate current pod service network:

        `kubeadm config print init-defaults`

        Look For:

        ```yaml
        networking:
          dnsDomain: cluster.local
          serviceSubnet: 10.96.0.0/12
        ```

    * Pull latest flannel network driver from interwebs and update pod network with it:

      `wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml`

    * Modify the Network to reflect serviceSubnet:

        ```yaml
          net-conf.json: |
          {
            "Network": "10.96.0.0/12",
            "Backend": {
              "Type": "vxlan"
            }
          }
      ```

    * Apply configuration:

      `kubectl apply -f kube-flannel.yaml`

    * Nodes should now be able to see one another.
  
9. Install MetalLB

