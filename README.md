# networkmesh
cilium and calico EKS
Cilium is a popular open-source networking and security solution that provides eBPF-based kernel-level features for container orchestration platforms, including Kubernetes. Integrating Cilium with Amazon Elastic Kubernetes Service (EKS) can enhance your cluster's networking capabilities, scaling, and security.

Here's a step-by-step guide to setting up Cilium with an EKS cluster:

### 1. Prerequisites

- An EKS cluster set up and configured.
- `kubectl` installed and configured to manage the EKS cluster.
- `helm` installed, a package manager for Kubernetes.

### 2. Create an IAM OIDC identity provider for the EKS cluster

This step allows the EKS cluster to create AWS Identity and Access Management (IAM) roles for Kubernetes service accounts.

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text
aws iam create-open-id-connect-provider --url <OIDC_URL> --thumbprint-list <THUMBPRINT> --client-id-list sts.amazonaws.com
```

Replace `<cluster-name>` with your EKS cluster's name, `<OIDC_URL>` with the OIDC URL obtained from the first command, and `<THUMBPRINT>` with the root CA thumbprint for the OIDC provider.

### 3. Install Cilium using Helm

First, add the Cilium Helm repository:

```bash
helm repo add cilium https://helm.cilium.io/
```

Next, create a namespace for Cilium:

```bash
kubectl create namespace cilium
```

Now you can install Cilium, specifying the version that is compatible with your EKS cluster version:

```bash
helm install cilium cilium/cilium --version 1.11.0 \
  --namespace cilium \
  --set eni=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set tunnel=disabled \
  --set nodeinit.enabled=true
```

The above command installs Cilium with ENI (Elastic Network Interface) mode, specific to AWS, enabling AWS-specific features.

### 4. Verify the Installation

To verify that Cilium has been installed correctly, you can check the Cilium pods:

```bash
kubectl -n cilium get pods
```

All the pods should be in the `Running` or `Completed` state.

### 5. Additional Configuration (Optional)

Depending on your use case, you may need to configure additional Cilium features, such as Hubble for observability, ClusterMesh for multi-cluster support, or custom network policies for enhanced security.

Refer to the [official Cilium documentation](https://docs.cilium.io/en/stable/gettingstarted/aws-eni/) for more details and advanced configuration options.

### Note

Make sure to follow the version compatibility between Cilium and your EKS cluster version. Always refer to the official Cilium documentation for the version that suits your cluster, as it might differ from the example provided here.

Cilium Network Policies provide granular control over the network communication between pods. They enable you to define rules for ingress and egress traffic, allowing or denying communication based on different parameters such as labels, protocols, ports, etc.

Below is an example of a Cilium Network Policy that illustrates how you can define security rules for your Kubernetes pods:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-frontend-to-backend"
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend # Apply this policy to endpoints with the label 'app: backend'
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend # Allow traffic from endpoints with the label 'app: frontend'
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP # Allow TCP traffic on port 80
```

This policy allows TCP traffic on port 80 from pods labeled with `'app: frontend'` to pods labeled with `'app: backend'`. Any other communication to the `'backend'` pods will be denied.

### How to Apply the Policy

To apply this policy to your Kubernetes cluster, save the YAML to a file, e.g., `cilium-policy.yaml`, and then use `kubectl` to apply it:

```bash
kubectl apply -f cilium-policy.yaml
```

### How to Validate the Policy

You can validate that the policy is applied correctly by using the `cilium` CLI or `kubectl`. Here's how you can list all Cilium Network Policies applied to the default namespace:

```bash
kubectl get ciliumnetworkpolicies -n default
```

### Conclusion

Cilium Network Policies offer powerful and flexible ways to control the network communication between your pods. This enables you to enhance the security of your applications by only allowing necessary connections. Always test your network policies in a development or staging environment first to ensure they behave as expected before deploying them to production.

Calico is another popular network and network policy provider that can be used with Kubernetes, including Amazon EKS. You can use Calico to enforce network policies across multiple EKS clusters. Here's an example of setting up Calico and applying a network policy across multiple EKS clusters.

### 1. Install Calico in Each EKS Cluster

First, you need to install Calico in each EKS cluster. You can follow the [official Calico AWS installation documentation](https://docs.projectcalico.org/getting-started/kubernetes/managed-public-cloud/aws) for detailed steps. Typically, it involves running:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Repeat this step in each EKS cluster where you want to use Calico.

### 2. Define a Global Network Policy

With Calico, you can define a global network policy that applies across all clusters. Here's an example policy that allows incoming TCP traffic on port 80 to any pod with the label `app: backend`, from any pod with the label `app: frontend`.

Save the following YAML as `global-policy.yaml`:

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  selector: app == 'backend'
  ingress:
    - action: Allow
      source:
        selector: app == 'frontend'
      protocol: TCP
      destination:
        ports:
          - 80
  egress:
    - action: Allow
```

### 3. Apply the Global Network Policy in Each Cluster

You'll need to apply this policy in each cluster where you want it to be enforced:

```bash
kubectl apply -f global-policy.yaml
```

### 4. Label Your Pods

Make sure the pods in your clusters have the corresponding labels `app: frontend` or `app: backend` as needed. You can add labels to existing pods by editing them or specify them in your deployment YAML files.

### Conclusion

By following these steps, you've set up Calico across multiple EKS clusters and created a global network policy that allows specific inter-cluster communication. You can now manage your network policies across all clusters through Calico.

Remember to refer to the [official Calico documentation](https://docs.projectcalico.org) for more detailed information on configuring and using Calico, including features like federation for more complex multi-cluster scenarios.
