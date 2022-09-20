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

    * Generate the certificate key

      `kubeadm certs certificate-key`

    * Print the join command

      `kubeadm token create --print-join-command`

    * Add the following flags to the above join command

      `--control-plane --certificate-key "Certificate-key from step 1"`

    * Command should look like below

      `sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07`

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

