# Service Mesh deployment demo

This project aims to demonstrate how to deploy a microservices architecture on AWS ECS, leveraging on AWS App Mesh for Lest/West integration between those services and on AWS API Gateway for North/South integration. 

The architecture of this deployment is using the following AWS Services:

1. [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html) : Used to deploy this whole architecture using the '*infrastructure as code*' paradigm.
1. [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-getting-started.html) : Used to define an isolated and secure network perimeter for the application. 
1. [Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/getting-started-ecs-ec2.html) : All the components of this application were encapsulated into containers. ECS is a container orchestration that aims to provide scalability and resiliency to a container-based architecture. 
1. [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancer-getting-started.html) : In order to provide internet-facing, easy access, to the frontend UI, an application load balancer was added to the solution.
1. [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancer-getting-started.html) : Used by API Gateway as its entry point into the VPC, and also load balancing the requests among the difference instances of each backend service. 
1. [AWS Cloud Map](https://docs.aws.amazon.com/cloud-map/latest/dg/setting-up-cloud-map.html) : Provides *service discovery* capabilities to the application. Since all these components were deployed using an auto-scaling topology using ephemeral containers, it is important to have a mechanism to allow each service to discover the location of other services. 
1. [AWS App Mesh](https://docs.aws.amazon.com/app-mesh/latest/userguide/appmesh-getting-started.html) : Creates a service mesh, allowing the microservices to talk to each other in a secure, reliable and scalable way, providing lest/west integration capabilities within the application.
1. [Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started.html) : Encapsulates access to the backend services, adding a layer of control to the application that way allowing north/south integration capabilities to the application.

> **DISCLAIMER**: This not a production ready system. It is meant only to educate in terms of what can be achieved with the components aforementioned. In order to understand AWS best practices to production topologies, please refer to [AWS Well Architected Framework](https://aws.amazon.com/architecture/well-architected/)

The application itself is very simple. It is composed by a Frontend Web UI written in [ruby](https://www.ruby-lang.org/en/) and two different backend services, one written in [nodejs](https://nodejs.org/en/) and another one written in [crystal](https://crystal-lang.org). 

![Application Architecture](./static/application-architecture.png) 

Their source code can be obtained here: 
- [nodejs](https://github.com/brentley/ecsdemo-nodejs)
- [crystal](https://github.com/brentley/ecsdemo-crystal)
- [frontend](https://github.com/brentley/ecsdemo-frontend)


## Topology

As mentioned earlier, this application will be deployed onto a ECS cluster, using a service mesh to encapsulate it. In the next diagram, you can notice that there are 2 different paths used reach the backend services, colored with different colors.

![alt](./static/architecture.png)

The blue path illustrates the lest/west scenario: The frontend UI is part of the service mesh, and it communicates with them directly within the mesh, relying on the Cloud Map service discovery to figure out the ip address where those services are running. 

The red path illustrates the north/south scenario. The fronted UI is deployed onto a separated VPC on the same AWS account, but it could be running on another AWS account or even on-premises. 

You can reach the frontend app of these scenarios by accessing its related application load balancer URL on your web browser.

## Communication

There are a few concepts hidden on the previous diagram that needs to be explored on each scenario.


### Blue scenario:

![alt](./static/dataflow-mesh.png)

- When the request arrives at the service mesh from the ALB, it reaches the ***Virtual Node*** endpoint declared for the frontend UI. A ***Virtual Node*** is an abstraction of a deployment or physical server where your service/application is running. In this case, we are only using one ***Virtual Node*** per service. It represents its deployment on ECS.  

- When the frontend UI makes a request to a backend service, it uses its ***Virtual Service*** name. Since containers are ephemeral, the only way that the frontend service has to know where the backend service is running, is by relying on a 'Service Discovery' component.    
&nbsp;  
Every time a new task is started at the ECS cluster, the ECS service register this task on AWS Cloud Map, and a DNS entry is created on Route 53 for this task. The ***Virtual Service*** is this name being used to register the task.

- Instead of reaching directly the ***Virtual Node*** (what could be done), the frontend UI is talking to the ***Virtual Router*** component. This router has the ability to abstract the connectivity to several distinct ***Virtual Nodes***, adding intelligence to the routing logic. For instance, if you want to setup a blue/green deployment of one of the backend services, you could add the routing rules on this component.  

### Red Scenario

![alt](./static/dataflow-apigw.png) 

- This scenario is using a 100% private api. Despite of the fact that API Gateway is a public AWS service, it has the ability to expose privately APIs only to authorized 
 VPCs, through the use of ***Interface Endpoints*** deployed onto those VPCs. This setup allows a private communication between the service consumer and the API itself. 

- For accessing private services on a AWS Account without having the need to expose them on the internet, API Gateway has the ability to leverage a ***VPC Link*** integration. This setup allows a private communication between the API and its backend services. 

- You can see that the backend services (nodejs, crystal) are the very same of the ones being accessed in the previous scenario. This setup demonstrates that a service lying on a service mesh can be exposed to consumers outside of the mesh as well.


## CloudFormation Stacks

For simplicity sake, this project was split into 3 Cloud Formation templates, each one being responsible for deploying part of the architecture.

1. **stack.yaml**  
&nbsp;  
Deploys the base stack of the architecture:  
	-	**Consumer** and **producer** VPCs and its related components
	-	1 **Application** Load Balancer on each VPC
	- 1 **Network** Load Balancer on the provider VPC
	-	1 ECS cluster on each VPC and its related components
	-	1 App Mesh instance on the provider VPC
	- 1 Cloud Map instance on the provider VPC
	- 1 API Gateway's private REST API 
	- API Gateway's VPC Link onto the **provider** VPC
	- API Gateway's Endpoint Interface onto the **consumer** VPC
2. **service.yaml**  
&nbsp;  
This template is used to instantiate an application/service on the mesh. It deploys the application as an ECS Task Definition onto the **provider** ECS cluster and its following related components:
	-	ECS Service
	-	Load Balancer listener 
	-	Load Balancer Target Group
	-	App Mesh Virtual Service
	-	App Mesh Virtual Node
	-	App Mesh Virtual Router and route
	-	Cloud Map service discovery entry
	-	2 CloudWatch log groups (one for the application itself, another one for its Envoy sidecar)

3.	**api.yaml**  
&nbsp;  
This template is used to instantiate the resources needed for the red scenario. It deploys Frontend application as a Task Definition onto the **consumer** ECS Cluster and the same related components related to it described on the previous template. In addition to that it deploys the following API Gateway components:
	-	Resource and Method for the nodejs backend
	-	Resource and Method for the crystal backend
	-	Stage
	- Deployment

### Getting Started

In order to deploy this topology on your AWS account, clone this repository and deploy each template in the order mentioned above. Remember that you need to deploy the service stack 3 times (one for each application). If you need detailed instructions, follow [this](./) guide.