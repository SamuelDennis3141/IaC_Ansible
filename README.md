# Infrastructure as Code using Ansible

![image](https://user-images.githubusercontent.com/112982429/188999151-1ccc0130-7a84-4670-acd1-0f9fccb25814.png)

### What is IaC?

Infrastructure as code (IaC) uses DevOps methodology and versioning with a descriptive model to define and deploy infrastructure, such as networks, virtual machines, load balancers, and connection topologies. Just as the same source code always generates the same binary, an IaC model generates the same environment every time it deploys.

IaC is a key DevOps practice and a component of continuous delivery. With IaC, DevOps teams can work together with a unified set of practices and tools to deliver applications and their supporting infrastructure rapidly and reliably at scale.

credit: https://docs.microsoft.com/en-us/devops/deliver/what-is-infrastructure-as-code

### What is Ansible?

Ansible can be used to provision the underlying infrastructure of a software's environment, virtualized hosts and hypervisors, network devices, and bare metal servers. It can also install services, add compute hosts, and provision resources, services, and applications inside of your cloud.

It provisions agent nodes with playbooks created on the controller node (which is the only node which must have ansible installed). The controller node can interact with any of the agent nodes without having ansible installed because it is an agentless service.

Playbooks are written in a language called YAML which uses a `.yml` extension.

credit: https://www.redhat.com/en/topics/automation/learning-ansible-tutorial

#### In this repository we will cover three different methods of deploying this infrastructure.

- [On Premesis](https://github.com/SamuelDennis3141/IaC_Ansible/tree/main/premesis)
- [Hybrid](https://github.com/SamuelDennis3141/IaC_Ansible/tree/main/hybrid)
- [Cloud](https://github.com/SamuelDennis3141/IaC_Ansible/tree/main/cloud) 
