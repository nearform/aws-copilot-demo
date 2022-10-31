# Fast infrastructure prototyping with AWS Copilot

There are no two same products we build, so the infrastructure requirements are not always the same. From the time when the business case is formed, we can start ideating on the solution design. Then, solution architects can start designing and prototyping the infrastructure depending on various constraints, including pre-existing platforms, different cloud providers, or a specific type of workload. Nowadays, deploying new workloads to the public cloud providers is already a standard, leading to a simplified architecting stage where architects can rely on the existing types of designs.

No matter what the final solution architecture is, there’re multiple best practices that we have to consider while setting up a new project, including highly available and reliable infrastructure, immutable environments, or having a proper deployment strategy supported by a solid continuous integration pipeline. However,  all those considerations take time to implement, can significantly lower the time to market, and also reduce the developer experience. As one of the key pillars of the DevOps philosophy is to shift responsibilities left (which means enabling the development team to work on the infrastructure definition and creation). The cognitive load needed for understanding the whole technology set used for a single project might be overwhelming. For that reason, having production-ready blueprints can really improve the teams’ velocity in the initial phase of the project build-up.


Amazon Web Services for example, provides a set of [opinionated solution designs](https://aws.amazon.com/solutions/implementations/aws-blueprints/) but also tools that can rapidly speed-up the creation of the most common infrastructure setups. One of the tools introduced in 2020 is [AWS Copilot CLI](https://github.com/aws/copilot-cli).

## AWS Copilot CLI

The AWS Copilot CLI is a tool for developers to build, release and operate production-ready containerized applications on AWS App Runner or Amazon ECS on AWS Fargate. It requires direct access to AWS for its explicit users, however, it’s also possible to define the fully-fledged pipelines that will take care of the infrastructure creation and application deployment.

Unlike typical Infrastructure as Code tools, AWS Copilot CLI depends heavely on a set of opinionated solutions, including:
* Request-Driven Web Service
* Load Balanced Web Service
* Worker Services

With the CLI you can not only scaffold the code needed to create the selected infrastructure but also analyze logs and health, debug and handle code deployment. Its simple but powerful set of options can guide you thru the process from having an empty repository to having an app deployed to the cloud.

The authors of AWS Copilot covered some fundamental architectures, however, they also helped to abstract away some not-so-basic concepts including networking or security. With great developer experience, naturally, there are trade-offs involved. The first and most important one is the fact, that once you will go off the script with your solution requirements, you have to dig into the weeds of the code being produced and used by the CLI. The moment you need to introduce additional cloud resources to your architecture you will be challenged by understanding AWS CloudFormation and how to write the templates. On the other hand, CloudFormation is the bread and butter for many cloud engineers and with the wide range of available open-source templates, the task might become trivial.


## AWS Copilot and DevOps best practises

Whatever the tooling, applying the DevOps philosophy principles should be a no-brainer for all the new green field projects, especially the ones deployed to the cloud. Depending on the software development lifecycle stage, engineers will put more attention to different principles. For example for the development environment, having highly available infrastructure does not always make sense (especially from the costs perspective), however, having functional pipelines for continuous integration and deployment should be a priority. 

AWS Copilot CLI has built-in support for creating the [pipelines](https://aws.github.io/copilot-cli/docs/manifest/pipeline/), which can be sufficient for small teams and simple use-case. As the CLI itself depends on the well-known construct of *infrastructure as code*, once the solution is designed, it generates manifest files that are later transformed into the AWS Cloudformation templates and stacks. It helps with the infrastructure immutability and can bring more confidence deploying the changes to production. 

When software engineers are involved with the infrastructure creation, the observability concept become a natural firt-class citizen from the day 1. It helps to facilitate a good logs management and support fostering infrastructure awareness across the whole team.

### Example AWS Copilot deployment

AWS is great with providing company-curated list of workshops that are ideal to get familar with some of the AWS Services. They also authored a simply AWS Copilot workshop, and that's the one we recommend you to [try it out](https://catalog.us-east-1.prod.workshops.aws/workshops/bbe33a01-4c98-4cfc-b859-d55d699419b8/en-US/10-what-we-are-building). The full list of available workshops can be found under [https://workshops.aws/](https://workshops.aws/). 

### Extending the out-of-the-box 

At NearForm we recognize the 3-tier architecture as one of the most common designs and our database of choice is an RDS with postgres. Copilot does not provide CLI-based flow to add this type of database therefore that's the first customisation we have to do, to start development. 

To add the database, first we need to understand how to add additional resources which is described in the [Copilot's docs](https://aws.github.io/copilot-cli/docs/developing/addons/modeling/). As the result we know there are 3 major steps involved:
* Create a Cloudformation template under `copilot/our_service/addons` directory.
* Create mandatory secrets for the db user name and password, for example with `copilot secret init`.
* Update the service manifest file with the secrets to be consumed as environment variables in our application.

The code inolved could look like this:
```yaml
# RDS-related template
Parameters:
  App:
    Type: String
    Description: Your application's name.
  Env:
    Type: String
    Description: The environment name your service, job, or workflow is being deployed to.
  Name:
    Type: String
    Description: The name of the service, job, or workflow being deployed.
Resources:
  # Subnet group to control where the DB gets placed
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Group of subnets to place DB into
      SubnetIds: !Split [ ',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PrivateSubnets' } ]
  # Security group to add the DB to the VPC,
  # and to allow the Fargate containers to talk to DB
  DatabaseSecurityGroup:
    Metadata:
      'aws:copilot:description': 'A security group to access the DB cluster'
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "DB Security Group"
      VpcId: { 'Fn::ImportValue': !Sub '${App}-${Env}-VpcId' }
  # Enable ingress from other ECS services created within the environment.
  DBIngress:
    Metadata:
      'aws:copilot:description': 'Allow ingress from containers in my application to the DB cluster'
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Ingress from Fargate containers
      GroupId: !Ref 'DatabaseSecurityGroup'
      IpProtocol: tcp
      FromPort: 5432
      ToPort: 5432
      SourceSecurityGroupId: { 'Fn::ImportValue': !Sub '${App}-${Env}-EnvironmentSecurityGroup' }
  # The cluster itself.
  DBInstance:
    Metadata:
      'aws:copilot:description': 'DB cluster'
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: postgres
      EngineVersion: '14.1'
      DBInstanceClass: 'db.t3.micro'
      AllocatedStorage: 8
      StorageType: gp2
      MultiAZ: false
      AllowMajorVersionUpgrade: false
      AutoMinorVersionUpgrade: true
      DeletionProtection: false
      BackupRetentionPeriod: 7
      EnablePerformanceInsights : false
      DBName: api
      MasterUsername: username
      MasterUserPassword: !Sub "{{resolve:ssm-secure:/copilot/${App}/${Env}/secrets/DATABASE_PASSWORD:1}}"
      DBSubnetGroupName: !Ref 'DBSubnetGroup'
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
  EndpointAddressParam:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub "/copilot/${App}/${Env}/secrets/DATABASE_ENDPOINT"
      Type: String
      Value: !GetAtt DBInstance.Endpoint.Address
Outputs:
  DatabaseSecurityGroup:
    Description: Security group for DB
    Value: !Ref DatabaseSecurityGroup
```

```yaml
secrets:                      
  DATABASE_USERNAME: /copilot/${COPILOT_APPLICATION_NAME}/${COPILOT_ENVIRONMENT_NAME}/secrets/DATABASE_USERNAME
  DATABASE_PASSWORD: /copilot/${COPILOT_APPLICATION_NAME}/${COPILOT_ENVIRONMENT_NAME}/secrets/DATABASE_PASSWORD
  DATABASE_ENDPOINT: /copilot/${COPILOT_APPLICATION_NAME}/${COPILOT_ENVIRONMENT_NAME}/secrets/DATABASE_ENDPOINT
```

Some of the variables are predefined by the Copilot runtime and you should get familiar with the basics of Cloudformation to fully understand the template. To see the full example see our [aws-copilot-demo](https://github.com/nearform/aws-copilot-demo) project at github.

## Trade offs and limitations

The AWS Copilot CLI can drastically speed up the creation of the development environments, while simultaneously it can become a bottleneck when developers will encounter some rough edges. As behind the scenes, the main IaC engine is Cloudformation, Copilot is affected by its limitations and problems. For example, if your first deployment will fail for any reason, you might be forced to remove the Cloudformation stack that is in the failed state manually. The moment you will need some modifications to the initial architectures, you must start using cloudformation templates as well as you must understand a bit more of the Copilot mechanics, including manifest syntax or the directory structure.


### Audience

The current market standard for defining the infrastructure as code is [Terraform](https://www.terraform.io/), but the landscape of infrastructure as code tools is still growing and their quality is still improving. Aside terreform, tools like AWS Cloudformation, AWS CDK are also very popular in the context of AWS. 

As AWS Copilot is an abstraction on top of AWS Clouformation, it is designed to ease the infrastructure creation for common patterns, sometimes allowing to ignore the complexity of underlying components. Therefore the tool seems a good choice for software engineers focused on developing business applications that want to be independent in an early stage of development from the wider cloud ecosystem, keeping the same or at least similar cloud components to the production-grade solution. Being able to experiment fast and deploy the solution as early as sprint 0 to the cloud, can help build confidence and get the customer’s feedback instantly.  At NearForm, depending on the needs, our DevOps specialists can either fine-tune and extend the AWS Copilot-based solution or simultaneously build a tailor-made infrastructure.

## Conclusion

