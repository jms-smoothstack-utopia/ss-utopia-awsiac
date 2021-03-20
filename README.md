<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Backend Infrastructure with AWS](#backend-infrastructure-with-aws)
  - [CloudFormation Template](#cloudformation-template)
    - [Diagram](#diagram)
  - [Additional Resources](#additional-resources)
    - [Using Existing resources](#using-existing-resources)
  - [Replicas and Auto-scaling](#replicas-and-auto-scaling)
  - [Stack Updates](#stack-updates)
  - [Stretch Goals](#stretch-goals)

<!-- /code_chunk_output -->


# Backend Infrastructure with AWS

This repository contains the files necessary to create the backend Utopia Airlines architecture in full. Using [Docker Compose  with ECS](https://docs.docker.com/cloud/ecs-integration/), we are able to write a simple [docker-compose.yaml](./docker-compose.yaml) file to stand up all necessary resources on AWS ECS with two simple commands:

```sh
docker context create ecs ss-utopia
docker compose --project-name ss-utopia --context ss-utopia up -d
```

This will parse and convert the template to a CloudFormation template and deploy it as a stack on AWS, taking __~80 lines of code__ and turning it into __>1,200 line CloudFormation template__. Using this tool significantly reduces the chance for error, automates a significant amount of boilerplate, and allows for a very simple deployment while still providing the benefits of a CloudFormation Stack.

From this:
![Docker Compose File](https://utopia-documentation-media.s3.amazonaws.com/compose-file.png)

To this:
![CloudFormation Template](https://utopia-documentation-media.s3.amazonaws.com/cfn-template.gif)

## CloudFormation Template
The created template can be examined before deployment by running the following command:

```sh
docker compose convert
```

This will display the created stack template that can be modified and run directly via the AWS CLI as necessary. The template is sent to `std:out` and can be piped into a file as necessary.

With Unix:
```sh
docker compose convert > stack.yaml
```

With PowerShell:
```powershell
docker compose convert | Out-File stack.yaml
```

### Diagram
Viewing the created template as a diagram inside AWS CloudFormation gives a clearer view of the complexity of this deployment:
![CFN Template Diagram](https://utopia-documentation-media.s3.amazonaws.com/template-diagram.png)

## Additional Resources
The Compose Specification for ECS allows additional resources to be created (or existing resources used) by including "overlays" to the compose file. In this architecture, we are including two overlays:

1. A certificate ARN for TLS Termination at the Network Load Balancer, allowing the frontend application to interface with the backend services over HTTPS and securing communication:

```yaml
    OrchestratorTCP443Listener:
      Properties:
        Certificates:
          - CertificateArn: "arn:aws:acm:us-east-1:247293358719:certificate/e82de04a-b93c-4ebf-a9e2-420805d1633d"
        Protocol: TLS
```

2. A Route 53 Record Set that points to the Network Load Balancer for simple DNS routing within the entire architecture.

```yaml
    Route53RS:
      Type: AWS::Route53::RecordSet
      Properties:
        Name: services.utopia-air.click
        Type: A
        AliasTarget:
          DNSName:
            Fn::GetAtt: LoadBalancer.DNSName
          HostedZoneId:
           Fn::GetAtt: LoadBalancer.CanonicalHostedZoneID
        HostedZoneId: Z07227751H35Z7E3SB7PD
```

### Using Existing resources
Using these overlays significantly reduces the developer effort to creating these resources and adding onto future iterations of the deployment. Currently, the template deploys within the default VPC and uses the default subnets. However, a stretch goal is to expand this template to include an entirely separate VPC network for the backend applications to live within. This will be accomplished by creating a separate network CFN template with outputs that can be referenced within this architecture to provide a fully secured application inside of private subnets.

## Replicas and Auto-scaling
The compose file specification already allows for basic replication strategies to be included as part of the deployment. Because this is a prototype proof of concept, this is not included in this template. However, it is a simple means of including an additional property to each service that requires replication:
```yaml
services:
  foo:
    deploy:
      replicas: 3
```

Auto-scaling is equally simple, allowing inline definitions to the service declaration:

```yaml
services:
  foo:
    deploy:
      x-aws-autoscaling:
        min: 1
        max: 10 #required
        cpu: 75
        # mem: - mutualy exlusive with cpu
```

These examples are credit to the [Deploying Docker containers on ECS](https://docs.docker.com/cloud/ecs-integration/#auto-scaling) documentation.

_Adding this behavior to the deployment is a stretch goal to be accomplished in future iterations following acceptance of the proof of concept._

## Stack Updates
Updating the existing stack is another process incredibly simplified by using Docker Compose. To make any modifications, update the existing [docker-compose.yaml](./docker-compose.yaml) file with any changes (such as image information, auto-scaling behavior, etc) and run the up command again
```sh
docker compose --project-name ss-utopia --context ss-utopia up -d
```
This will create a change stack and apply it to CloudFormation directly. Resources that can be modified without downtime will be modified as appropriate. For resources requiring replacement, this will be done on a rolling basis according to a configuration written in a `deploy.update_config` file.

_Automating deployment updates to this stack via a CI/CD pipeline with Jenkins is a stretch goal._

## Stretch Goals
For future iterations, this project will be improved with the following:
- [ ] Creating an external, private network stack for the services.
- [ ] Adding replication/auto-scaling as necessary based on stress test feedback.
- [ ] Automating stack updates via the Jenkins CI/CD pipeline.