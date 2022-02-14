# 从EC2启动模板创建EKS节点组

### 1、创建EKS集群

1.1 指定eks集群使用的已有vpc和子网，至少需要2个不同可用区的私有子网

```
VPC_ID=vpc-0df4d4463a499e046
VPC_CIDR=10.0.0.0/16
PUBLIC_01=subnet-02ed9b13b581b4306
PUBLIC_02=subnet-06bc2b917a38cfe05
PRIVATE_01=subnet-09bc6ae338cdbb9e2
PRIVATE_02=subnet-0b96323814e58f8cb
```
1.2 给子网打标签，以便后续使用

```
aws ec2 create-tags --resources ${PRIVATE_SUBNET_01} --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources ${PRIVATE_SUBNET_01} --tags Key=kubernetes.io/role/internal-elb,Value=1
aws ec2 create-tags --resources ${PRIVATE_SUBNET_02} --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources ${PRIVATE_SUBNET_02} --tags Key=kubernetes.io/role/internal-elb,Value=1

aws ec2 create-tags --resources ${PUBLIC_SUBNET_01} --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources ${PUBLIC_SUBNET_01} --tags Key=kubernetes.io/role/elb,Value=1
aws ec2 create-tags --resources ${PUBLIC_SUBNET_02} --tags Key=kubernetes.io/cluster/eksworkshop,Value=shared
aws ec2 create-tags --resources ${PUBLIC_SUBNET_02} --tags Key=kubernetes.io/role/elb,Value=1
```
1.3 设置环境变量

```
AWS_REGION=cn-northwest-1
CLUSTER_NAME=containerd
NG_NAME=demo
INSTANCE_TYPE=r5a.4xlarge
KEY_PAIR=kp_eks_01
```
1.4 创建EC2密钥对

```
aws ec2 create-key-pair --region ${AWS_REGION} --key-name $KEY_PAIR
```
1.5 创建生成eks集群配置文件

```
cat << EOF > test-cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
  version: "1.21"

vpc:
  id: ${VPC_ID}
  cidr: ${VPC_CIDR}
  subnets:
    public:
      public-01: 
        id: ${PUBLIC_01}
      public-02:
        id: ${PUBLIC_02}
    private:
      private-01:
        id: ${PRIVATE_01}
      private-02:
        id: ${PRIVATE_02}

managedNodeGroups:
  - name: demo
    instanceType: $INSTANCE_TYPE
    minSize: 1
    maxSize: 3
    desiredCapacity: 1
    volumeSize: 50
    volumeType: gp3
    privateNetworking: true
    ssh:
      allow: true
      publicKeyName: $KEY_PAIR
EOF
```
1.6 创建EKS集群，包含1个节点组

```
eksctl create cluster -f test-cluster-config.yaml
```

### 2、定制EKS节点使用的EC2启动模板

2.1 获取节点组的启动模板数据，并保存为json文件，实例ID从控制台中获取

```
aws ec2 get-launch-template-data --instance-id $INSTANCE_ID --query 'LaunchTemplateData' > template.json
```
2.2 参照示例中的template-001.json对该json文件进行修改

```
{
    "BlockDeviceMappings": [
        {
            "DeviceName": "/dev/xvda",
            "Ebs": {
                "Encrypted": false,
                "DeleteOnTermination": true,
                "SnapshotId": "snap-048696411e6808658",
                "VolumeSize": 50,
                "VolumeType": "gp3",
                "Throughput": 125
            }
        }
    ],
    "NetworkInterfaces": [
        {
            "DeleteOnTermination": true,
            "Description": "aws-K8S-i-0b904aec9d3e4b708",
            "DeviceIndex": 1,
            "Groups": [
                "sg-05c3980fcabf44f78",
                "sg-09f5e4161df64bd31"
            ],
            "InterfaceType": "interface",
            "NetworkCardIndex": 0
        },
        {
            "DeleteOnTermination": true,
            "Description": "aws-K8S-i-0b904aec9d3e4b708",
            "DeviceIndex": 2,
            "Groups": [
                "sg-05c3980fcabf44f78",
                "sg-09f5e4161df64bd31"
            ],
            "InterfaceType": "interface",
            "NetworkCardIndex": 0
        },
        {
            "DeleteOnTermination": true,
            "Description": "",
            "DeviceIndex": 0,
            "Groups": [
                "sg-05c3980fcabf44f78",
                "sg-09f5e4161df64bd31"
            ],
            "InterfaceType": "interface",
            "NetworkCardIndex": 0
        }
    ],
    "ImageId": "ami-00b85dc6b0adb960b",
    "InstanceType": "r5a.large",
    "KeyName": "zhy_linux",
    "UserData": "TUlNRS1WZXJzaW9uOiAxLjAKQ29udGVudC1UeXBlOiBtdWx0aXBhcnQvbWl4ZWQ7IGJvdW5kYXJ5PSIvLyIKCi0tLy8KQ29udGVudC1UeXBlOiB0ZXh0L3gtc2hlbGxzY3JpcHQ7IGNoYXJzZXQ9InVzLWFzY2lpIgojIS9iaW4vYmFzaApzZXQgLWV4CkI2NF9DTFVTVEVSX0NBPUxTMHRMUzFDUlVkSlRpQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENrMUpTVU0xZWtORFFXTXJaMEYzU1VKQlowbENRVVJCVGtKbmEzRm9hMmxIT1hjd1FrRlJjMFpCUkVGV1RWSk5kMFZSV1VSV1VWRkVSWGR3Y21SWFNtd0tZMjAxYkdSSFZucE5RalJZUkZSSmVVMUVTWGhOVkVFelRYcFplVTlXYjFoRVZFMTVUVVJKZDA5VVFUTk5lbGw1VDFadmQwWlVSVlJOUWtWSFFURlZSUXBCZUUxTFlUTldhVnBZU25WYVdGSnNZM3BEUTBGVFNYZEVVVmxLUzI5YVNXaDJZMDVCVVVWQ1FsRkJSR2RuUlZCQlJFTkRRVkZ2UTJkblJVSkJUa2hHQ2pselZrZFdORXhYVDA1UmNuZzBjRVZIZUVsRVFrbDZjRXRzVmxGdVJrNVNhRk5xTm5WVFMyVmtNVWh2Vm5SRVlrRk5hV0ZhZEdkQ1VGTTBjemhoZGpnS0swYzJNbkYxVEROWk1GWjRPREF6ZEhsUmRFWkZLekZDTUV4d2VWSldZbVZSV0RCSVMyUnZObXhvZVRZNFdFeEllREphVFV4eFIwWkZWMnhFYkRCTlJRcDJVVW80ZFZaeGFDODBjV0ZrWWxOcVRYWnViRmRsTHk5NFZHdExja1o2VjJKeU1rdEJTVEZuYTBsTE5WWjNTRFpyVjFSalRFTnhkMVJKVTFnNVJqRnZDbk56YlVReE5DOUtiREJpYWpCMVNYRklVaTlEVWxWd2IzVlJWQzkyY0RRdk4ySlhPVmh0V2pOa05WaE9la0Y2YjB0TlFVcFlOemQ2VEhSNlFXbERSV01LVDFGTGFsWXZaWHBQUTBaTlF5dGlWVGhEUkhsTk1HazJkalkzUW5SalNHRTJaR3NyYUVsdlRsVldOMmxMT1hOdlFsZDVaMDlWYmpkR1RtNHlVM0F4U3dwRmNrZHVaMW8zZG1OamRXMHpNWEZGZUdGalEwRjNSVUZCWVU1RFRVVkJkMFJuV1VSV1VqQlFRVkZJTDBKQlVVUkJaMHRyVFVFNFIwRXhWV1JGZDBWQ0NpOTNVVVpOUVUxQ1FXWTRkMGhSV1VSV1VqQlBRa0paUlVaSU4wdzFWazFNUTJoV1FtZzBVa2hwY2pWU1YxTk9kbGd2VUdsTlFUQkhRMU54UjFOSllqTUtSRkZGUWtOM1ZVRkJORWxDUVZGQmVGUXpNRGg2UzI4d0swUmlSbWRwWms4elFXZERkMFZ6VlZGNVkyb3hPR0pZZFV0RldrcFJLMmt5VVVwb01YRldPUXBYZG5reWJrbEhWbFZRVFZRek5IYzFTaXRtWWxKUk5FZFlZMnQzWjJWeVFUWldVVGRxUjBwamJVbDVhMnR2ZEhoRllYSmpUbUl4U1hjMFFYSjFjek5QQ2xSR1YyWmxTMWMyUkhkbFdubHVNakJtU205UVRtMDRjMmhPUkVzNVl5OU5ibTVPU1ZRMFJqZDVSV2hZY0RGWVExZEpSR2R2Um5oR1RYaHVaV3hxTlRFS1VETkRlV3AyYzNacVRqVkZTekY0VkRkYU4weDRjVEJEUm5BMlQyTnlZa1JXY210bGNERm1kR3d5UVRkcmFtMUlTMVJMV25OT1ZscHNNbXR2Ym1ZMWRncFNTRVptUjNRMFEwbHFObWczZFhkTFZHaHljRGhPTUZKblVXOWlUeXNyVUdWS1ZrY3JZVVZ3YkU1a1RXdFdSREl3Y2pCT04xRXZjelYwVDFkWE0yeHBDbU41TVVSVFQweExVa3haVWxOR1MyRnhUR2RTYzAxWllWbDRhRmNyU0ZGSU4wSlNUQW90TFMwdExVVk9SQ0JEUlZKVVNVWkpRMEZVUlMwdExTMHRDZz09CkFQSV9TRVJWRVJfVVJMPWh0dHBzOi8vNjlCREFGNDhCODdCNjFCQzBGNjA5M0JCMzY1MEM2RjAuZ3I3LmNuLW5vcnRod2VzdC0xLmVrcy5hbWF6b25hd3MuY29tLmNuCks4U19DTFVTVEVSX0ROU19JUD0xNzIuMjAuMC4xMAovZXRjL2Vrcy9ib290c3RyYXAuc2ggY29udGFpbmVyZCAtLWt1YmVsZXQtZXh0cmEtYXJncyAnLS1ub2RlLWxhYmVscz1la3MuYW1hem9uYXdzLmNvbS9zb3VyY2VMYXVuY2hUZW1wbGF0ZVZlcnNpb249MSxhbHBoYS5la3NjdGwuaW8vY2x1c3Rlci1uYW1lPWNvbnRhaW5lcmQsYWxwaGEuZWtzY3RsLmlvL25vZGVncm91cC1uYW1lPWRlbW8sZWtzLmFtYXpvbmF3cy5jb20vbm9kZWdyb3VwLWltYWdlPWFtaS0wMGI4NWRjNmIwYWRiOTYwYixla3MuYW1hem9uYXdzLmNvbS9jYXBhY2l0eVR5cGU9T05fREVNQU5ELGVrcy5hbWF6b25hd3MuY29tL25vZGVncm91cD1kZW1vLGVrcy5hbWF6b25hd3MuY29tL3NvdXJjZUxhdW5jaFRlbXBsYXRlSWQ9bHQtMDQ2NTAxY2M4NjM2NzVmMTAnIC0tYjY0LWNsdXN0ZXItY2EgJEI2NF9DTFVTVEVSX0NBIC0tYXBpc2VydmVyLWVuZHBvaW50ICRBUElfU0VSVkVSX1VSTCAtLWRucy1jbHVzdGVyLWlwICRLOFNfQ0xVU1RFUl9ETlNfSVAKCi0tLy8tLQ==",
    "TagSpecifications": [
        {
            "ResourceType": "instance",
            "Tags": [
                {
                    "Key": "eks:cluster-name",
                    "Value": "containerd"
                },
                {
                    "Key": "alpha.eksctl.io/nodegroup-type",
                    "Value": "managed"
                },
                {
                    "Key": "alpha.eksctl.io/nodegroup-name",
                    "Value": "test"
                },
                {
                    "Key": "Name",
                    "Value": "containerd-test-Node"
                },
                {
                    "Key": "eks:nodegroup-name",
                    "Value": "test"
                },
                {
                    "Key": "kubernetes.io/cluster/containerd",
                    "Value": "owned"
                }
            ]
        }
    ],
    "MetadataOptions": {
        "HttpPutResponseHopLimit": 2
    }
}
```

2.3 从该启动模板数据创建新的启动模板

```
aws ec2 create-launch-template \
  --launch-template-name TemplateForTest01 \
  --launch-template-data file://template-001.json
```
2.4 在控制台中修改启动模板 - Containerd

* 存储（卷），添加卷2，设备名称选择“/dev/xvdb”，大小为500GiB，卷类型选择“st1”
* 高级详细信息 - 用户数据，在最后的“--//--”前加入如下内容：

```
echo "Running custom user data script"
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data

sudo cp /etc/fstab /etc/fstab.orig
uuid=`sudo blkid | grep nvme1n1 | cut -d " " -f2 | cut -d "=" -f2 | sed 's/\"//g'`
result="UUID=${uuid}  /data  xfs  defaults,nofail  0  2"
sudo sed -i '$a\'"$result"'' /etc/fstab

/etc/eks/bootstrap.sh containerd --container-runtime containerd

sudo mkdir /data/containerd
sudo sed -i "s#/var/lib#/data/root#g" /etc/containerd/config.toml

```
最后，点击“创建模板版本”，例如设置为版本“2”

2.5 在控制台中修改启动模板 - Docker

* 存储（卷），添加卷2，设备名称选择“/dev/xvdb”，大小为500GiB，卷类型选择“st1”
* 高级详细信息 - 用户数据，在最后的“--//--”前加入如下内容：

```
echo "Running custom user data script"
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data
sudo mount /dev/nvme1n1 /data

sudo cp /etc/fstab /etc/fstab.orig
uuid=`sudo blkid | grep nvme1n1 | cut -d " " -f2 | cut -d "=" -f2 | sed 's/\"//g'`
result="UUID=${uuid}  /data  xfs  defaults,nofail  0  2"
sudo sed -i '$a\'"$result"'' /etc/fstab

sudo mkdir /data/docker
sudo sed -i '/^ExecStart=/{s#$# --graph /data/docker#}' /usr/lib/systemd/system/docker.service
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```
最后，点击“创建模板版本”，例如设置为版本“2”

### 3、从启动模板创建节点组

3.1 从控制台获取启动模板ID

```
LAUNCH_TEMPLATE_ID=xxx
```
3.2 从启动模板创建节点组

```
cat << EOF > test-ng-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: $CLUSTER_NAME
  region: $AWS_REGION

managedNodeGroups:
  - name: $NG_NAME
    minSize: 1
    maxSize: 4
    desiredCapacity: 2
    privateNetworking: true
    launchTemplate:
      id: $LAUNCH_TEMPLATE_ID
      version: "2"
EOF
```
3.3 从启动模板创建节点组

```
eksctl create ng --config-file=test-ng-config.yaml
```
### 4、删除旧的节点组

执行如下命令，删除用作模板的旧节点组

```
eksctl delete ng demo --cluster $CLUSTER_NAME
```
