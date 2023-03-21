Instalando aws-cli e cdk

Para instalar o aws-cli, vamos baixar o arquivo contido no [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), bem como seguir o passo a passo orientado.

Após instalado, devemos criar um IAM user no console do aws, cadastrando um access key e secret key. Tendo essas informações,
Devemos executar o seguinte comando:

```
aws configure
```

Forneça os dados e estará configurado.

Será necessário também instalar o cdk. Para isso, precisamos instalar, além do Java, as seguintes ferramentas:
* [node.js](https://nodejs.org/en/download/)
* [Binário do Maven](https://maven.apache.org/download.cgi)

Com relação ao binário do Maven, precisamos, após descompactar o conteúdo em um diretório, salvar esse caminho numa
variável de sistema chamada MAVEN_HOME. E também inserir na variável path %MAVEN_HOME%\bin.

Feito esses passos, executar:

```
 npm install -g aws-cdk
```

Após isso, executar o comando abaixo:
```
cdk bootstrap
```

## Criando VPC

Podemos listar as VPCs existentes com o comando abaixo:

```
cdk list
```

No nosso caso, apresentou **Vpc**, que era a VPC cadastrada na aplciação no projeto Java

```
cdk init app --language java
```

Então, para fazer o deploy, executamos:
```
cdk deploy Vpc
```


### Criação do VPC
```
public class VpcStack extends Stack {

    private Vpc vpc;
    public VpcStack(final Construct scope, final String id) {
        this(scope, id, null);
    }

    public VpcStack(final Construct scope, final String id, final StackProps props) {
        super(scope, id, props);

        vpc = Vpc.Builder.create(this, "vpc01")
                .maxAzs(3)
                .build();
    }

    public Vpc getVpc() {
        return vpc;
    }
}
```

### Criação de Clusters

```
public class ClusterStack extends Stack {

    private Cluster cluster;
    public ClusterStack(final Construct scope, final String id, Vpc vpc) {
        this(scope, id, null, vpc);
    }

    public ClusterStack(final Construct scope, final String id, final StackProps props, Vpc vpc) {
        super(scope, id, props);

        cluster = Cluster.Builder.create(this, id)
                .clusterName("cluster-01")
                .vpc(vpc)
                .build();
    }

    public Cluster getCluster() {
        return cluster;
    }
}
```
Podemos observar que a criação do cluster tem dependência com a Vpc. 


### Criação de Service

Abaixo podemos ver a criação do servicde através da criação do objeto ApplicationLoadBalancedFargateService, 
juntamente com características do serviço, como porta, nome, recursos, etc.
```
ApplicationLoadBalancedFargateService service01 = ApplicationLoadBalancedFargateService.Builder.create(this, "ALB01")
                .serviceName("service-01")
                .cluster(cluster)
                .cpu(512)
                .memoryLimitMiB(1024)
                .desiredCount(2)
                .listenerPort(8080)
                .taskImageOptions(
                        ApplicationLoadBalancedTaskImageOptions.builder()
                                .containerName("aws-project01")
                                .image(ContainerImage.fromRegistry("fabiobione/curso_aws_project01:1.0.0"))
                                .containerPort(8080)
                                .logDriver(LogDriver.awsLogs(AwsLogDriverProps.builder()
                                        .logGroup(LogGroup.Builder.create(this, "Service01LogGroup")
                                                .logGroupName("Service01")
                                                .removalPolicy(RemovalPolicy.DESTROY)
                                                .build())
                                        .streamPrefix("Service01")
                                        .build()))
                                .build()
                )
                .publicLoadBalancer(true)
                .build();
```

O Trecho abaixo serve para configurar o healthcheck, que escuta o ednpoint exposto pelo spring actuator.
```
        service01.getTargetGroup().configureHealthCheck(new HealthCheck.Builder()
                .path("/actuator/health")
                .port("8080")
                .healthyHttpCodes("200")
                .build());
```

Já abaixo veremos as configurações de auto scaling:
```
        scalableTaskCount.scaleOnCpuUtilization("Service01AutoScaling", CpuUtilizationScalingProps.builder()
                .targetUtilizationPercent(58)
                .scaleInCooldown(Duration.seconds(60))
                .scaleOutCooldown(Duration.seconds(60))
                .build());
```

Na nossa classe principal, ficará assim:
```
    public static void main(final String[] args) {
        App app = new App();

        VpcStack vpcStack = new VpcStack(app, "Vpc");
        ClusterStack clusterStack = new ClusterStack(app, "Cluster", vpcStack.getVpc());
        clusterStack.addDependency(vpcStack);

        Service01Stack service01Stack = new Service01Stack(app, "Service01", clusterStack.getCluster());
        service01Stack.addDependency(clusterStack);
        app.synth();
    }
```
Nota-se que o cluster tem uma dependência com a Vpc, e o service dependência com o cluster.

Dessa forma, podemos executar o comando abaixo
para ver as diferenças desde o último deploy:
```
cdk deploy Vpc Cluster Service01
```

Após essa execução, o serviço estará disponível na url gerada no outuput do comando.