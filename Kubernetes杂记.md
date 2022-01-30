##### 控制器：
pod是kubernetes的最小控制单位，是对容器的进一步抽象和封装，在对容器进行组合的基础上又添加了更多的属性和字段，使得更轻松地操作它。操作这些pod的逻辑则由控制器完成，比较重要的有对pod状态、数量的控制，比如声明的deployment中通过指定replicas的数量，deployment则会一直控制对应的replicas的数量在声明的期待值。
控制器之所以被统一放在k8s项目pkg/controller目录下，就是因为它们都遵循Kubernetes项目中的一个通用编排模式，即：控制循环（control loop）。控制循环的普遍逻辑是在自己的生命周期内（通常由用户控制，maybe infinite loop）获取当前控制对象的实际状态，通过与声明期待状态对比，确认是否需要执行编排动作及动作内容。具体实现中，实际状态旺旺来自于kubernetes集群本身；而期待状态，一般来自用户提交的yaml文件。编排过程叫做调谐（reconcile）。通过定义控制器的api暴露函数，实现自定义的控制器（如deployment）是kubernetes“面向api对象编程”的体现。提交的yaml中上半部分提供了控制器的定义，加上下半部分被控制对象的模板组成。
##### 无状态应用（Deployment）：
##### StatefulSet：
把真实世界的应用状态抽象了两种情况，**拓扑状态**（意味多实例间不完全对等的关系，应用实例必须按照某些顺序启动，比如应用的主节点A先于节点B启动。如果删除AB节点也是按顺序删除。还有如新老pod的网络标识需要一致，这样访问者才能用相同的方法访问到新pod）、**储存状态**（比如数据库应用的多个储存实例）。
##### 声明式API与Kubernetes编程范式：
所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”（这个 YAML 文件其实就是一种“声明”），表示所期望的最终状态是什么样子就可以了。而如果提交的是一个个命令，去指导怎么一步一步达到期望状态，这就是“命令式”了。“命令式 API”接收的请求只能一个一个实现，否则会有产生冲突的可能；“声明式API”一次能处理多个写操作，并且具备 Merge 能力。
在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。通过这样的结构，整个Kubernetes里的所有API对象，实际上就可以用如下的树形结构表示出来：
声明式API与Kubernetes编程范式：
所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”（这个 YAML 文件其实就是一种“声明”），表示所期望的最终状态是什么样子就可以了。而如果提交的是一个个命令，去指导怎么一步一步达到期望状态，这就是“命令式”了。“命令式 API”接收的请求只能一个一个实现，否则会有产生冲突的可能；“声明式API”一次能处理多个写操作，并且具备 Merge 能力。
在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。通过这样的结构，整个Kubernetes里的所有API对象，实际上就可以用如下的树形结构表示出来：

声明式yaml的第一、第二项是描述APIVersion（Group/Version）和kind（Resource）分别描述api对象资源类型、组和版本。当我们提交这个yaml后，kubernetes会解析yaml文件内容转换成一个kubernetes的一个对象。kubernetes通过匹配api对象的组（对于Pod、Node等是不需要group，他们的group是“”），然后进一步匹配到api对象的版本号，最后会匹配api对象的资源类型。

在发起创建k8s对象的post请求后，编写的yaml信息会提交给apiserver。而apiserver首先会过滤请求（如授权、超时处理、审计等），然后进入MUX和Routes流程（MUX和Routes是apiserver完成URL和Handler绑定的场所）通过上一段流程找到对应对象类型定义，接着apiserver更具对象类型定义使用用户使用的yaml文件里的字段创建一个该类型对象，接下来apiserver会精心Admission和Validation操作（Validation负责校验这个对象的各个字段是否合法。这个被校验过的api对象都保存在了apiserver一个叫做Registry数据结构中，只要一个api对象定义能在Registry中查到，他就是一个有效的Kubernetes api对象。），最后apiserver会把验证过的API对象转换成用户最初提交的版本，进行序列化操作，并调用Etcd的API把它保存起来。
CRD的全称是Custom Resource Definition。顾名思义，它指的就是，允许用户在Kubernetes中添加一个跟Pod、Node类似的、新的API资源类型，即：自定义API资源。写的类似pod模板yaml是叫做CR，里面定义的具体的数值是在描述一个具体的“自定义API资源”实例，而让k8s认识这个CR就需要CRD的宏观定义。
定义一个网络的CRD文件：
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```
可以看到，在这个CRD中，我指定了“group: samplecrd.k8s.io”“version: v1”这样的API信息，也指定了这个CR的资源类型叫作Network，复数（plural）是networks。然后，我还声明了它的scope是Namespaced，即：我们定义的这个Network是一个属于Namespace的对象，类似于Pod。这就是一个Network API资源类型的API部分的宏观定义了。接下来，我还需要让Kubernetes“认识”这种YAML文件里描述的“网络”部分，比如“cidr”（网段），“gateway”（网关）这些字段的含义。
自定义资源项目结构：
```
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── register.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```
pkg/apis/samplecrd就是API组的名字，v1是版本，而v1下面的types.go文件里，则定义了Network对象的完整描述。[link](https://github.com/resouer/k8s-controller-custom-resource)在pkg/apis/samplecrd目录下创建了一个register.go文件，用来放置后面要用到的全局变量，在pkg/apis/samplecrd目录下添加一个doc.go文件（Golang的文档源文件），定义在doc.go文件的注释，起到的是全局的代码生成控制的作用。接下来，我需要添加types.go文件。顾名思义，它的作用就是定义一个Network类型到底有哪些字段（比如，spec字段里的内容）。最后，我需要再编写一个pkg/apis/samplecrd/v1/register.go文件。在前面对APIServer工作原理的讲解中，我已经提到，“registry”的作用就是注册一个类型（Type）给APIServer。其中，Network资源类型在服务器端注册的工作，APIServer会自动帮我们完成。但与之对应的，我们还需要让客户端也能“知道”Network资源类型的定义。这就需要我们在项目里添加一个register.go文件。
