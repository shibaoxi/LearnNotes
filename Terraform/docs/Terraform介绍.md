# Terraform 教程

## Terraform概述

### Terraform 客户端安装

Ubuntu用户

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository -y "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install -y terraform
```

CentOS用户

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```

Mac用户

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

对于Windows用户，官方推荐的包管理器是choco，可以去<https://chocolatey.org/> 下载安装好chocolatey后，以管理员身份启动powershell，然后：

```bash
choco install terraform
```

### Terraform 介绍

Terraform是HashiCorp公司旗下的Provision Infrastructure产品, 是AWS APN Technology Partner与AWS DevOps Competency Partner。Terraform是一个IT基础架构自动化编排工具，它的口号是“Write, Plan, and Create Infrastructure as Code”, 是一个“基础设施即代码”工具，类似于AWS CloudFormation，允许您创建、更新和版本控制的AWS基础设施。

Terraform基于AWS Go SDK进行构建，采用HashiCorp配置语言（HCL）对资源进行编排，具体的说就是可以用代码来管理维护IT资源，比如针对AWS，我们可以用它创建、修改或删除 S3 Bucket、Lambda,、EC2、Kinesis、VPC等各种资源。并且在真正运行之前可以看到执行计划(即干运行-dryrun)。由于状态保存到文件中，因此能够离线方式查看资源情况（前提是不要在 Terraform 之外对资源进行修改）。Terraform 配置的状态除了能够保存在本地文件中，也可以保存到 Consul, S3等处。

Terraform是一个高度可扩展的工具，通过Provider来扩展对新的基础架构的支持，几乎支持所有的云服务平台，AWS只是Terraform内建 Providers 中的一种。

在Terraform诞生之前，我们对AWS资源的操作主要依赖Console、AWS CLI、SDK或Serverless。AWS CLI什么都能做，但它是无状态的，必须明确用不同的命令来创建、修改和删除。Serverless不是用来管理基础架构的，用Lambda创建资源是很麻烦的事。AWS提供的CloudFormation，虽然功能非常强大，但是大量的JSON代码阅读困难。

### Terraform 基础概念

#### Provider

Terraform被设计成一个多云基础设施编排工具，不像CloudFormation那样绑定AWS平台，Terraform可以同时编排各种云平台或是其他基础设施的资源。Terraform实现多云编排的方法就是Provider插件机制。

<img src="https://i.loli.net/2021/08/24/VuSxmtFf2YjOWck.png" width=600 />

Terraform使用的是HashiCorp自研的go-plugin库(<https://github.com/hashicorp/go-plugin>)，本质上各个Provider插件都是独立的进程，与Terraform进程之间通过rpc进行调用。Terraform引擎首先读取并分析用户编写的Terraform代码，形成一个由data与resource组成的图(Graph)，再通过rpc调用这些data与resource所对应的Provider插件；Provider插件的编写者根据Terraform所制定的插件框架来定义各种data和resource，并实现相应的CRUD方法；在实现这些CRUD方法时，可以调用目标平台提供的SDK，或是直接通过调用Http(s) API来操作目标平台。

我们在编写完成代码后，执行 terraform init。 terraform init 会分析代码中所用到Provider，并尝试下载Provider插件到本地。
下载的插件一般会存放在项目文件夹下面的.terraform文件夹。

##### 搜索Provider

下载地址：<https://registry.terraform.io/browse/providers>

##### Provider的声明

一组Terraform代码要被执行，相关的Provider必须在代码中被声明。不少的Provider在声明时需要传入一些关键信息才能被使用.

```json
terraform {
    required_providers{
        aws = {
            source  = "hashicorp/aws"
            version = "~> 3.0"
        }
    }
}

provider "aws" {
    region = "cn-north-1"
    access_key = "my-access-key"
    secret_key = "my-secret-key"
}
```

在这段Provider声明中，首先在terraform节的required_providers里声明了本段代码必须要名为aws的Provider才可以执行，source = "hashicorp/aws"这一行声明了aws这个插件的源地址(Source Address)。一个源地址是全球唯一的，它指示了Terraform如何下载该插件。

required_providers中的插件声明还声明了该源码所需要的插件的版本约束，在例子里就是version = ">=1.24.1"。Terraform插件的版本号采用MAJOR.MINOR.PATCH的语义化格式，版本约束通常使用操作符和版本号表达约束条件，条件之间可以用逗号拼接，表达AND关联，例如">= 1.2.0, < 2.0.0"。可以采用的操作符有：

- =(或者不加=，直接使用版本号)：只允许特定版本号，不允许与其他条件合并使用
- !=：不允许特定版本号
- >,>=,<,<=：与特定版本号进行比较，可以是大于、大于等于、小于、小于等于
- ~>：锁定MAJOR与MINOR，允许PATCH号大于等于特定版本号，例如，~>0.9等价于>=0.9, <1.0，\~>0.8.4等价于>=0.8.4, <0.9

推荐使用">="操作符约束最低版本。如果你是在编写旨在由他人复用的模块代码时，请避免使用"~>"操作符，即使你知道模块代码与新版本插件会有不兼容。

##### 多个Provider实例

我们也可以声明多个同类型的Provider

```json
terraform {
    required_providers{
        azurerm = {
            source = "hashicorp/azurerm"
            version = ">=2.46.0"
        }
        aws = {
            source  = "hashicorp/aws"
            version = "~> 3.0"
        }
    }
}

provider "aws" {
    region = "cn-north-1"
    access_key = "my-access-key"
    secret_key = "my-secret-key"
}

provider "azurerm" {
    features {
      
    }
}
```

#### 状态管理

当我们成功地执行了一次terraform apply，创建了期望的基础设施以后，我们如果再次执行terraform apply，生成的新的执行计划将不会包含任何变更，Terraform会记住当前基础设施的状态，并将之与代码所描述的期望状态进行比对。第二次apply时，因为当前状态已经与代码描述的状态一致了，所以会生成一个空的执行计划。

简单来说，Terraform将每次执行基础设施变更操作时的状态信息保存在一个状态文件中，默认情况下会保存在当前工作目录下的terraform.tfstate文件里。

我们在terraform apply之后立即再次apply是不会执行任何变更的，那么如果我们删除了这个tfstate文件，然后再执行apply会发生什么呢？Terraform读取不到tfstate文件，会认为这是我们第一次创建这组资源，所以它会再一次创建代码中描述的所有资源。更加麻烦的是，由于我们前一次创建的资源所对应的状态信息被我们删除了，所以我们再也无法通过执行terraform destroy来销毁和回收这些资源，实际上产生了资源泄漏。所以妥善保存这个状态文件是非常重要的。

另外，如果我们对Terraform的代码进行了一些修改，导致生成的执行计划将会改变状态，那么在实际执行变更之前，Terraform会复制一份当前的tfstate文件到同路径下的terraform.tfstate.backup中，以防止由于各种意外导致的tfstate损毁。

如果我们是一个团队在使用Terraform管理一组资源，团队成员之间要如何共享这个状态文件？能不能把tfstate文件签入源代码管理工具进行保存？

把tfstate文件签入管代码管理工具是非常错误的，这就好比把数据库签入了源代码管理工具，如果两个人同时签出了同一份tfstate，并且对代码做了不同的修改，又同时apply了，这时想要把tfstate签入源码管理系统可能会遭遇到无法解决的冲突。

为了解决状态文件的存储和共享问题，Terraform引入了远程状态存储机制，也就是Backend。Backend是一种抽象的远程存储接口，如同Provider一样，Backend也支持多种不同的远程存储服务：

- 标准：支持远程状态存储与状态锁
- 增强：在标准的基础上支持远程操作(在远程服务器上执行plan、apply等操作)

目前增强型Backend只有Terraform Cloud云服务一种。

状态锁是指，当针对一个tfstate进行变更操作时，可以针对该状态文件添加一把全局锁，确保同一时间只能有一个变更被执行。不同的Backend对状态锁的支持不尽相同，实现状态锁的机制也不尽相同，例如consul backend就通过一个.lock节点来充当锁，一个.lockinfo节点来描述锁对应的会话信息，tfstate文件被保存在backend定义的路径节点内；s3 backend则需要用户传入一个Dynamodb表来存放锁信息，而tfstate文件被存储在s3存储桶里。名为etcd的backend对应的是etcd v2，它不支持状态锁；etcdv3则提供了对状态锁的支持，等等等等。
