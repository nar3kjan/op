### Enabling GPU Slicing in Amazon EKS for AI Workloads
This guide provides instructions for setting up GPU slicing for AI workloads on Amazon EKS.

## Step 1: Label GPU Nodes

First, identify the nodes in your EKS cluster that have GPU capabilities.

```bash
kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
```

Label the GPU node(s) appropriately to schedule GPU workloads specifically on them.

```bash
kubectl label node <your-gpu-node-name> eks-node=gpu
```

### Step 2: Deploy the NVIDIA GPU Device Plugin
Next, install the NVIDIA GPU device plugin to expose the GPU resources to Kubernetes, allowing better management and scheduling of these resources. It's possible to use helm for deployment.

Create a values.yaml if necessary for specifying any configurations and use helm to install the plugin.

https://github.com/sanjeevrg89/eks-gpu-sharing-demo/blob/main/nvdp-values.yaml

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install --generate-name nvdp/nvidia-device-plugin -n kube-system -f nvdp-values.yaml
```

Check if the device plugin is running:

```bash
kubectl get daemonset -n kube-system | grep nvidia
```

### Step 3: Enable Time-Slicing

Configure time-slicing by creating a ConfigMap to specify the slicing strategy, which slices the GPU resources into virtual slices that can be allocated to different pods.

```bash
cat << EOF > nvidia-device-plugin-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin
  namespace: kube-system
data: 
  config.json: |  
    {
      "migStrategy": "none",
      "sharing": {
        "timeSlicing": {
          "resources": [
            {
              "name": "nvidia.com/gpu",
              "replicas": 10
            }
          ]
        }
      }
    }
EOF

kubectl apply -f nvidia-device-plugin-config.yaml
```
### Step 4: Update NVIDIA Device Plugin with Time-Slicing

Upgrade the existing NVIDIA device plugin to use the new configuration for time-slicing.

```bash
helm upgrade nvdp nvdp/nvidia-device-plugin -n kube-system -f nvdp-values.yaml --set config.name=nvidia-device-plugin --reuse-values --force
```

### Step 5: Verify Node’s GPU Capacity
Check that the node’s GPU capacity is updated to reflect the virtual GPUs:

```bash
kubectl get nodes -o json | jq -r '.items[] | select(.status.capacity."nvidia.com/gpu" != null) | {name: .metadata.name, capacity: .status.capacity}'
```

### Step 6: Deploy Workloads
Deploy a deployment or pods that request a fraction of a GPU.

Example deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
   name: sample-gpu-deployment
spec:
   replicas: 20
   template:
     spec:
       containers:
       - name: cuda-container
         image: nvidia/cuda:11.0-base
         resources:
           limits:
             nvidia.com/gpu: "0.1"  # requesting 10% of a GPU
```
            
Apply this configuration and verify:

```bash
kubectl apply -f deployment.yaml
kubectl get pods -n gpu-demo
```

### With Karpenter
For auto-scaling with Karpenter while using GPU slicing:

Follow Steps for installing Nvidia plugin from the previous section.

### Configure Node Pool in Karpenter
Update or create your NodePool configuration with instance details:

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: "node.kubernetes.io/instance-type"
          operator: In
          values: ["p3.8xlarge"]
      nodeClassRef:
        name: default
```


By following these steps, you enable the time-slicing of GPU resources on Amazon EKS, which allows for more efficient utilization of GPU capacity in scenarios where tasks require GPUs intermittently or don’t utilize a full GPU. This setup also enhances cost optimization by allowing greater concurrency in GPU usage.