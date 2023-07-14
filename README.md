# Terraform code to spin a K8s cluster using AWS EC2 instances

The Terraform code here will build a Kubernetes cluster using AWS EC2 instances and kubeadm.

## 1 - Infrastructure

The environment is built on AWS using Terraform. If you want to become more familiar with Terraform, it's time! : ) You will need an AWS account and Terraform installed on your computer.

1. Start by cloning this repository:

```bash
git clone https://github.com/regisftm/aws-ec2-k8s-tf
```

2. Change the directory to Terraform, and run the Terraform initialization:

```bash
cd aws-ec2-k8s-tf/terraform
terraform init
```

3. Edit the `variables.tf` file and change accordingly. The default value will generate 1 EC2 instance type t3.small for the control-plane and 1 EC2 instance type t3.medium for the worker node. The AWS region selected is `ca-central-1`. Feel free to change the variable values to whatever you want in your environment. I can't promise that it will work well if you use smaller instance types.

```bash
vi variables.tf
```

4. Apply the Terraform code. This code will build the EC2 instances and install Kubernetes and other software used in this demonstration.

```bash
terraform apply --auto-approve
```
5. After a few minutes, you will see the output containing the created public IPs for the EC2 instances. 

<pre>
Apply complete! Resources: 12 added, 0 changed, 0 destroyed.

Outputs:

control_plane_public_ip = "3.96.49.113"
workers_public_ips = {
  "worker-01" = "3.99.20.164"
}
</pre>

Go to the next step to finalize the **Kubernetes Cluster Configuration** and configure & install the Calico CNI.

## 2 - Kubernetes Cluster Configuration

After installing kubeadm, kubectl, and kubelet on both nodes and initializing the control-plane node with kubeadm, the next step is to join the worker node(s) to the cluster. Follow the instructions below to complete this process:

1. The Terraform generated and saved a key pair in the `terraform` folder. Utilize this key to establish an SSH connection with the control-plane node and retrieve the kubeadm join command. Locate this command in the `/var/log/cloud-init-output.log` file.

   ```bash
   ssh -i ec2-nodes-key ubuntu@<control_plane_public_ip_address>
   ```

   ```bash
   grep "kubeadm.*discovery\|discovery.*kubeadm" /var/log/cloud-init-output.log
   ```
   
   The output will be something like:
   
   <pre>
   kubeadm join <control_plane_private_ip>:6443 --token 9lbxla.pjsptj0m9wra8tyi --discovery-token-ca-cert-hash sha256:bfd99111c1f98dcb4ec225d2ec56fee13d2207057a2811eb67b217be8330c6ed
   </pre>

2. Using the same key located in the `owasp-toronto/terraform` folder, open another terminal and ssh to the worker node (if you have more than one, repeat these steps for all of them).

   ```bash
   ssh -i owasp-key ubuntu@<worker_node_public_ip_address>
   ```
   ```bash
   sudo su - root
   ```

   Paste the `kubeadm join` command copied from the control-plane node.

   The output will look like the following:

   <pre>
   root@worker-01:~# kubeadm join 172.31.44.20:6443 --token 92ap7u.vwmkiesc0cjcdphp --discovery-token-ca-cert-hash sha256:d60463cc14666f454579eca7c26b61569b90da4d75aa912a293529f49194d50a
   [preflight] Running pre-flight checks
   [preflight] Reading configuration from the cluster...
   [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
   [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
   [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
   [kubelet-start] Starting the kubelet
   [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
   
   This node has joined the cluster:
   * Certificate signing request was sent to apiserver and a response was received.
   * The Kubelet was informed of the new secure connection details.
   
   Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
   
   root@worker-01:~#
   </pre>

   After joining the worker node to the cluster, you can close the terminal connected to the worker node.

3. From the terminal connected to the control-plane, verify if the node successfully joined the cluster by running the following command as `root` (use `sudo su - root`):

   ```bash
   sudo su - root
   kubectl get nodes
   ```

   The output should be:

   <pre>
   NAME            STATUS     ROLES           AGE     VERSION
   control-plane   Ready      control-plane   14m     v1.26.5
   worker-01       Ready      &lt;none&gt;          2m36s   v1.26.5
   </pre>

### Congratulation, you did it! Now go enjoy your Kubernetes cluster!

---

## Clean up

Go to the Terraform folder at `aws-ec2-k8s-tf/terraform` and use the command below.

```bash
terraform destroy --auto-approve
```
