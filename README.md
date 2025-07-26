### 一、EKS是什么
EKS（Elastic Kubernetes Service）是亚马逊云（AWS）提供的托管式Kubernetes服务。它的主要作用是帮助用户在AWS环境中轻松运行Kubernetes集群，而无需自行管理Kubernetes的控制平面（Master节点）。其优势包括：
 - **简化集群管理**：AWS负责控制平面的维护、监控和升级等工作，用户只需专注于部署应用。
 - **集成AWS服务**：无缝对接AWS的各种服务，如EC2（计算资源）、EBS（存储）、IAM（身份验证和访问管理）等。
 - **高可用性和可扩展性**：自动在多个可用区部署控制平面，确保高可用性；支持动态扩展集群节点以应对不同的工作负载需求。

### 二、部署前的环境配置
1. **拥有AWS账号**：如果没有，前往[AWS官网](https://aws.amazon.com/)注册。注册过程中需要提供一些基本信息和支付方式（用于付费服务，部署简单应用可能费用较低或在免费套餐内） 。
2. **安装和配置AWS CLI（命令行界面）**：
    - **下载安装**：根据操作系统，从[AWS CLI官方文档](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)获取安装包并安装。例如在Linux系统中，可以使用包管理器安装（如`sudo apt install awscli` ，具体命令因发行版而异）；在Windows系统中，可通过MSI安装文件进行安装。
    - **配置凭证**：运行`aws configure`命令，按照提示输入AWS访问密钥（Access Key ID）、秘密访问密钥（Secret Access Key）、默认区域（如`us-west-2` ，可根据需求选择）和输出格式（一般选择`json` ）。这些凭证可以在AWS管理控制台的IAM用户设置中创建和获取。
3. **安装kubectl**：它是Kubernetes的命令行工具，用于与Kubernetes集群进行交互。
    - **下载安装**：从[Kubernetes官方文档](https://kubernetes.io/docs/tasks/tools/)获取对应操作系统的安装包。在Linux系统中，常见的安装方式如下：
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
    - **验证安装**：安装完成后，运行`kubectl version --client` ，如果显示版本信息，则说明安装成功。
4. **安装eksctl**：eksctl是用于创建和管理EKS集群的命令行工具。
    - **下载安装**：在Linux系统中，可使用如下命令安装：
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
    - **验证安装**：运行`eksctl version` ，若显示版本号，说明安装成功。

### 三、使用EKS创建Kubernetes集群
1. **网络规划（参考视频https://www.bilibili.com/video/BV11yeMeyEUC ）**：
    - **VPC（虚拟私有云）**：每个EKS集群都需要一个VPC。可以使用已有的VPC，也可以创建新的VPC。如果创建新VPC，要规划好IP地址范围（如`10.0.0.0/16` ） 。
    - **子网划分**：
        - **公有子网**：用于部署需要公网访问的资源，如负载均衡器。每个可用区至少要有一个公有子网。
        - **私有子网**：用于部署不需要直接公网访问的资源，如应用的后端服务。同样，每个可用区至少要有一个私有子网。在私有子网中部署节点时，需要配置NAT网关（Network Address Translation），使节点能够访问互联网以拉取容器镜像等资源。
2. **创建EKS集群**：使用eksctl创建集群是较为便捷的方式，命令如下：
```bash
eksctl create cluster \
--name my-eks-cluster \
--region us-west-2 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--managed
```
    - **参数解释**：
        - `--name`：指定集群的名称，这里为`my-eks-cluster` ，可自行定义。
        - `--region`：指定集群所在的AWS区域，如`us-west-2` 。
        - `--nodegroup-name`：指定节点组的名称，`standard-workers`是示例名称。
        - `--node-type`：指定节点的EC2实例类型，`t3.medium`是一种通用型实例，可根据应用需求选择。
        - `--nodes`：指定初始节点数量为3个。
        - `--nodes-min`和`--nodes-max`：分别指定节点数量的最小值和最大值，用于自动扩展，这里最小1个，最大4个。
        - `--managed`：表示使用AWS托管的节点组，AWS会负责节点的管理和维护。

### 四、部署简单应用
1. **编写应用的Deployment YAML文件**：以部署Nginx为例，创建一个名为`nginx-deployment.yaml`的文件，内容如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
    - **内容解释**：
        - `apiVersion`：指定Kubernetes API的版本。
        - `kind`：指定资源类型为`Deployment` 。
        - `metadata.name`：Deployment的名称为`nginx-deployment` 。
        - `spec.replicas`：指定要创建的Pod副本数量为3个。
        - `spec.selector.matchLabels`：用于选择要管理的Pod，这里选择标签为`app: nginx`的Pod。
        - `spec.template.metadata.labels`：为Pod定义标签，方便选择和管理。
        - `spec.template.spec.containers`：定义容器的相关信息。
        - `spec.template.spec.containers.name`：容器名称为`nginx` 。
        - `spec.template.spec.containers.image`：使用`nginx:1.14.2`的镜像。
        - `spec.template.spec.containers.ports`：指定容器暴露的端口为80。
2. **部署应用**：在包含`nginx-deployment.yaml`文件的目录下，运行如下命令：
```bash
kubectl apply -f nginx-deployment.yaml
```
这条命令会根据YAML文件的定义，在EKS集群中创建3个Nginx Pod。
3. **验证应用部署**：
    - **查看Pod状态**：运行`kubectl get pods` ，如果看到3个状态为`Running`的Nginx Pod，则说明部署成功。
    - **访问应用**：为了能够访问Nginx应用，需要创建一个Service。创建一个名为`nginx-service.yaml`的文件，内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```
    - **内容解释**：
        - `apiVersion`和`kind`：指定API版本和资源类型为`Service` 。
        - `metadata.name`：Service的名称为`nginx-service` 。
        - `spec.type`：指定Service的类型为`LoadBalancer` ，这会在AWS中创建一个弹性负载均衡器。
        - `spec.ports`：定义Service暴露的端口和目标Pod的端口。
        - `spec.selector`：选择标签为`app: nginx`的Pod作为后端服务。
    - **创建Service**：运行`kubectl apply -f nginx-service.yaml` 。
    - **获取访问地址**：运行`kubectl get services` ，找到`nginx-service`对应的`EXTERNAL-IP` ，在浏览器中访问该IP地址，应该能够看到Nginx的欢迎页面。

### 五、后续管理与维护
1. **监控集群和应用**：可以使用工具如Prometheus和Grafana来监控EKS集群和应用的性能指标，如CPU使用率、内存使用率、网络流量等。具体配置步骤较为复杂，涉及到在集群中部署Prometheus和Grafana的相关组件，并进行数据源配置等操作。
2. **更新应用**：如果需要更新应用，例如更新Nginx的版本，可以修改`nginx-deployment.yaml`文件中的`image`字段，然后重新运行`kubectl apply -f nginx-deployment.yaml`命令，Kubernetes会自动更新Pod中的镜像。
3. **扩展应用**：如果应用的负载增加，可以通过修改`nginx-deployment.yaml`中的`spec.replicas`字段来手动扩展Pod数量；也可以通过创建HorizontalPodAutoscaler（HPA）来实现自动扩展。创建一个名为`nginx-hpa.yaml`的文件，内容如下：
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
    - **内容解释**：
        - `apiVersion`和`kind`：指定API版本和资源类型为`HorizontalPodAutoscaler` 。
        - `metadata.name`：HPA的名称为`nginx-hpa` 。
        - `scaleTargetRef`：指定要扩展的目标Deployment。
        - `minReplicas`和`maxReplicas`：分别指定Pod数量的最小值和最大值。
        - `metrics`：指定用于自动扩展的指标，这里以CPU使用率为指标，当平均CPU使用率达到50%时，自动扩展Pod数量。
    - **部署HPA**：运行`kubectl apply -f nginx-hpa.yaml` ，Kubernetes会根据CPU使用率自动调整Nginx Pod的数量。 
