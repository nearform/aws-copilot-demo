# Sprint 0 infrastructure with AWS Copilot

Each product we build is unique, the same as the infrastructure requirements. From the time the business case is formed, we can start ideating on how to address the needs. Then, solution architects can start designing and prototyping the infrastructure depending on various constraints, including pre-existing platforms, different cloud providers or a specific type of workload.

Nowadays, deploying new workloads to public cloud providers is already a standard, leading to a simplified architecting stage where architects can rely not only on their expertise but also on existing design patterns.

No matter what the final solution architecture is, there are some best practices that we need to consider while setting up a new project, including highly available and reliable infrastructure, immutable environments or having a proper deployment strategy supported by a solid continuous integration pipeline.

All these considerations take time to implement, can significantly lower the time to market and also poorly affect developer experience.

One of the key pillars of the DevOps philosophy is to shift responsibilities left. One aspect of shifting left is enabling the development team to work on infrastructure definition and creation. However, the cognitive load needed to understand the whole technology set used for a single project might be overwhelming. For that reason, having production-ready blueprints can improve teams’ velocity in the initial phase of the project build-up.

Amazon Web Services, for example, provides a set of [opinionated solution designs](https://aws.amazon.com/solutions/implementations/aws-blueprints/) but also tools that can rapidly speed up the creation of the most common infrastructure setups. One of the tools introduced in 2020 is [AWS Copilot CLI](https://github.com/aws/copilot-cli).

## AWS Copilot CLI

The AWS Copilot CLI is a tool for developers to build, release and operate production-ready containerized applications on AWS App Runner or Amazon ECS on AWS Fargate. It requires direct access to AWS for its explicit users, however, it’s also possible to define the fully-fledged pipelines that will take care of the infrastructure creation and application deployment.

Unlike typical Infrastructure as Code tools, AWS Copilot CLI depends heavily on a set of opinionated solutions, including:

- Request-Driven Web Service
- Load Balanced Web Service
- Worker Services

With the CLI you can not only scaffold the code needed to create the selected infrastructure but also analyze logs/health, debug and handle code deployment. It has a simple but powerful set of options which can guide you through the process from an empty repository to an app being deployed to the cloud.

The authors of AWS Copilot covered some fundamental architectures, however, they also helped to abstract some not-so-basic concepts including networking or security.

With great developer experience, naturally, there are trade-offs involved. The first and most important one is the fact that once you go off script with your solution requirements, you have to dig into the weeds of the manifest code being produced and used by the CLI.

The moment you need to introduce additional cloud resources to your architecture you will be challenged to understand AWS CloudFormation and how to write the templates. On the other hand, CloudFormation is still a popular tool in the AWS ecosystem and with the wide range of available open-source templates, the task might become trivial.

## AWS Copilot and DevOps best practices

Whatever the tooling, applying the DevOps philosophy principles should be a no-brainer for all new green field projects, especially ones deployed to the cloud. Depending on the software development lifecycle stage, engineers will pay more attention to different principles. For example for the development environment, having highly available infrastructure does not always make sense (especially from a cost perspective), however, having functional pipelines for continuous integration and deployment should be a priority.

AWS Copilot CLI has built-in support for creating the [pipelines](https://aws.github.io/copilot-cli/docs/manifest/pipeline/), which can be sufficient for small teams and simple use-case. As the CLI itself depends on the well-known construct of _infrastructure as code_, it generates manifest files that are later transformed into the AWS CloudFormation templates and stacks. This means the infrastructure can be created in an immutable fashion and can bring more confidence in deploying the changes to production as each environment is easily reproducible.

Another positive side effect of shifting responsibilities left happens when software engineers are involved with infrastructure creation, the observability concept becomes a natural first-class citizen from day 1. It helps to facilitate good log management and support fostering infrastructure awareness across the whole team.

### Audience

The current market standard for defining infrastructure as code (as well as a NearForm's standard) is [Terraform](https://www.terraform.io/).

While Terraform may be the de facto standard, there are a wide range of other infrastucture as code tools available. For many cloud engineers tools like [AWS CloudFormation](https://aws.amazon.com/cloudformation/), and [AWS CDK](https://aws.amazon.com/cdk/) are also very popular in the context of AWS.

As AWS Copilot is an abstraction on top of AWS CloudFormation, it is designed to ease infrastructure creation for common patterns. Therefore the tool can be a good choice for software engineers focused on developing business applications that need to be independent from the wider cloud ecosystem in early development stages by keeping things simple and using the same or at least similar cloud components to the production-grade solution.

Being able to experiment fast and deploy the solution, as early as sprint 0, to the cloud can help build customer confidence and enables teams to get instantaneous customer feedback. At NearForm, depending on the needs, our [DevOps specialists](https://www.nearform.com/services/devops-platform-engineering/) can either fine-tune and extend the AWS Copilot-based solution and/or simultaneously build a tailor-made infrastructure.

## Trade-offs and limitations

The AWS Copilot CLI can drastically speed up the creation of the development environments.

It can also become a bottleneck when developers encounter some rough edges.

Behind the scenes, the main IaC engine is CloudFormation and Copilot is affected by its limitations. For example, if your first deployment fails for any reason, you may be forced to remove the CloudFormation stack that is in the failed state manually.

If you need to modify the initial architectures, you must start using CloudFormation templates and you'll need a basic understanding of the Copilot mechanics, including manifest syntax or the directory structure.

Copilot's generated IAM policies don't always follow the most fine-grained approach for granting access. You might need to improve the overall security posture of the solution.

### Example AWS Copilot deployment

AWS is great with providing a company-curated list of workshops that are ideal to get familiar with some of the AWS Services. They also authored a simple AWS Copilot workshop, and that's the one we recommend you [try out](https://catalog.us-east-1.prod.workshops.aws/workshops/bbe33a01-4c98-4cfc-b859-d55d699419b8/en-US/10-what-we-are-building). The full list of available workshops can be found under [https://workshops.aws/](https://workshops.aws/).

### Extending the out-of-the-box

At NearForm we recognize the 3-tier architecture as one of the most common designs. The missing piece from the infrastructure template generated by AWS Copilot is the database - our database of choice is an RDS with Postgres.

Copilot does not provide CLI-based flow to add this type of database.

This is the first customization we need to implement in order to enable the development team to begin work. The CloudFormation code presented below can be written by software engineers with an interest in this space or by your cloud engineers or other IaC specialists in your organization.

To add the database, first, we need to understand how to add additional resources. This topic is described in [Copilot's docs](https://aws.github.io/copilot-cli/docs/developing/addons/modeling/). Long story short, there are 3 major steps involved:

- Create a CloudFormation template under `copilot/our_service_name/addons` directory.
- Create mandatory secrets for the database user name and database password (`copilot secret init`).
- Update the service manifest file with the secrets to be consumed as environment variables in our application.

The code involved could look like this:

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
      SubnetIds:
        !Split [',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PrivateSubnets' }]
  # Security group to add the DB to the VPC,
  # and to allow the Fargate containers to talk to DB
  DatabaseSecurityGroup:
    Metadata:
      'aws:copilot:description': 'A security group to access the DB cluster'
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: 'DB Security Group'
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
      SourceSecurityGroupId:
        { 'Fn::ImportValue': !Sub '${App}-${Env}-EnvironmentSecurityGroup' }
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
      EnablePerformanceInsights: false
      DBName: api
      MasterUsername: username
      MasterUserPassword: !Sub '{{resolve:ssm-secure:/copilot/${App}/${Env}/secrets/DATABASE_PASSWORD:1}}'
      DBSubnetGroupName: !Ref 'DBSubnetGroup'
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
  EndpointAddressParam:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub '/copilot/${App}/${Env}/secrets/DATABASE_ENDPOINT'
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

Some of the variables are predefined by the Copilot runtime and you should familiarize yourself with the basics of CloudFormation to fully understand the template. To see the full example see our [aws-copilot-demo](https://github.com/nearform/aws-copilot-demo) project on GitHub.

Once the call is in place, you can simply run `copilot svc deploy`. This will create a CloudFormation changeset and will apply the changes to your infrastructure.

Since the creation of the database is not instant, be ready to wait around 15 to 20 minutes to have your stack updated. The above command will not only create a database but will update your ECS configuration with the environment variables that you can use in your code to connect to the database.

In your terminal window you should see similar results:

```bash
✔ Proposing infrastructure changes for stack copilot-demo-dev-api
- Creating the infrastructure for stack copilot-demo-dev-api                      [create complete]    [899.5s]
  - An Addons CloudFormation Stack for your additional AWS resources              [create complete]    [531.3s]
    - Allow ingress from containers in my application to the DB cluster           [create complete]    [0.0s]
    - DB cluster                                                                  [create complete]    [341.5s]
    - A security group to access the DB cluster                                   [create complete]    [4.6s]
  ...
  ...
  - An ECS service to run and maintain your tasks in the environment cluster      [create complete]    [318.6s]
    Deployments
               Revision  Rollout      Desired  Running  Failed  Pending
      PRIMARY  2         [completed]  1        1        0       0
  - A target group to connect the load balancer to your service                   [create complete]    [2.1s]
  - An ECS task definition to group your containers and run them on ECS           [create complete]    [2.7s]
  - An IAM role to control permissions for the containers in your tasks           [create complete]    [21.7s]
✔ Deployed service api.
```

## Conclusion

With public cloud offerings becoming more stable and mature the tools are following the same path. Cloud providers are paying more attention to developer experience and they are actively improving the tooling ecosystem based on community feedback.

To make the tools more user-friendly, authors must include some boundaries for the generic nature of the tools and quite often they need to introduce quite an opinionated approach.

Although the generic vs opinionated vs complex dependency is not linear, stakeholders should accept the trade-offs introduced by different tools. It's important for stakeholders to analyze and carefully choose the right tool for the job.

AWS Copilot is one of the tools that should be taken into consideration when quick prototyping and developer experience are at the top of the priority list.
