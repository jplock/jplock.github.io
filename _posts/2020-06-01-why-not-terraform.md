layout: page
title: "Why not Terraform?"
date: 2020-06-01 12:00:00 +4000
categories: aws terraform

TL;DR - [Terraform](https://terraform.io/) is an excellent tool, but I didn't know it as well as [AWS CloudFormation](https://aws.amazon.com/cloudformation/). CloudFormation was the quickest way for me to demonstrate the art of the possible with AWS and how it can fundamentally change an organization.

Every week, for the past few months, as [The Hartford](https://www.thehartford.com/) has been progressing in our cloud journey, I get asked "why not Terraform?" and I explain our usage of AWS CloudFormation in our cloud platform. The short answer is I think Terraform is an excellent tool, I just personally didn't know it well enough, and I felt AWS had come a long way for the need to introduce an outside tool.

For some background, I began to heavily use AWS at my [previous job](https://www.fireeye.com/) in 2013. AWS was rough around the edges and didn't have nearly the amount of services and capabilities that it does today, especially with a multi-account strategy. I've watched AWS mature and grow over the years and have continued to be impressed by the quantity of the services being delivered, the pace at which they are released, and the way they integrate with one another.

When I decided to move to The Hartford, I had the opportunity to help a large financial enterprise begin to invest in the AWS platform as they intended - with a developer-first mindset and let them focus on the "undifferenced heavy lifting". I had the opportunity to think, "if I were going to set up a large enterprise for scale, embracing cloud-native solutions as much as possible, what technologies would I choose to accomplish that goal?" Terraform was the obvious choice and there are numerous companies doing this today; using Terraform to automate all of the pieces of provisioning AWS accounts, networks, and services that AWS didn't already do.

2020 is a lot different than 2013 though...7 years is a lifetime in the cloud space, was there an easier solution? I feel pretty strongly, and I think AWS would agree with me, that a lot of what people had previously been doing with Terraform can now be accomplished using AWS itself. Our platform makes use of the following services, which I'll go into more detail:

- [AWS Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html)
- [AWS SSO Permission Sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
- [AWS CloudFormation StackSets for Organizations](https://aws.amazon.com/about-aws/whats-new/2020/02/aws-cloudformation-stacksets-introduces-automatic-deployments-across-accounts-and-regions-through-aws-organizations/)
- [AWS CloudFormation Macros](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-macros.html)
- [AWS CloudFormation Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [AWS Step Functions](https://aws.amazon.com/step-functions/)
- [AWS Lambda](https://aws.amazon.com/lambda/)
- [AWS Transit Gateway Network Manager](https://aws.amazon.com/transit-gateway/network-manager/) (and AWS Direct Connect)
- [AWS EventBridge](https://aws.amazon.com/eventbridge/)
- [AWS Service Catalog](https://aws.amazon.com/servicecatalog/)

AWS Control Tower is a relatively new service (GA on 6/24/2019) that automates the account creation process. It does this by creating an account and associating it to an AWS Organizations Organizational Unit (OU) which has pre-defined Service Control Policies (SCPs) already applied. These policies provide various security guardrails to ensure the newly created account adheres to the required security controls.

CloudFormation StackSets for Organizations recognizes that a new account has been added to the OU and proceeds to apply the stack set template to the account, which varies depending on the OU. These CloudFormation templates create base IAM service roles, KMS keys for AWS CodePipeline, SNS topics for alerting, S3 buckets for logging and build artifacts, and AWS Systems Manager (SSM) Parameters.

After the account has been created, Control Tower sends a lifecycle event through EventBridge that initiates an AWS Step Function state machine. The state machine executes individual Lambda functions in parallel within the account:

- Deletes the default VPC from all regions (Control Tower does this as well, this primarily verifies the deletion)
- Accepts a centralized AWS Service Catalog portfolio of reference architectures that is shared to the organization, and associates principals within the account to the portfolio so they can provision products
- Blocks public S3 bucket access at the account level
- Adds a CloudWatch Logs resource policy for Route53 access logging in the account

The step function is used to automate anything that we can't perform natively with CloudFormation.

We now have a new AWS account with all of the necessary security controls in place and we can grant users access into the account using AWS SSO Permission Sets.

Depending on the role the user has access to, they'll be able to provision pre-approved reference architectures from AWS Service Catalog. These reference architectures are as small as a single resource (an S3 bucket) or as large as a full front-end web stack (CloudFront + S3 + Lambda@Edge + Route53). We also have products to allow users to provision AWS CodePipelines to programatically maintain their own portfolios of products ([Inception](https://www.imdb.com/title/tt1375666/) anyone?). Users can also provision themselves a VPC from the Service Catalog that provides private connectivity back to our data centers.

The VPC product in the Service Catalog makes use of a CloudFormation Custom Resource to integrate with our IP Address Management (IPAM) platform. The custom resource queries the IPAM for the next available subnet range and then returns control back to CloudFormation to continue with the VPC provisioning. The CloudFormation template is then transformed with a CloudFormation Macro that handles all of the subnet splitting and route table creation for the VPC, along with creating an AWS Transit Gateway Attachment resource.

AWS Transit Gateway (GA on 11/26/2018), in combination with AWS Direct Connect (GA 8/3/2011), allows an organization to have connectivity to AWS regions over private connections into a central network gateway. We have our Transit Gateway and Direct Connects in a separate isolated "network" AWS account that VPCs in all of the other accounts peer with (by setting up attachments). We were able to build automation with AWS Transit Gateway Network Manager (which only lives in us-west-2 by the way...) that automatically accepts the attachment request and enables route propagation into the VPCs as they are created (using EventBridge and a cross-region Lambda function).

Overall, I've been very impressed with the amount of automation we've been able to achieve using AWS-native services. We do have some custom Lambda functions to maintain, all of which can be tested, are well understood and are isolated in their usage. If we were to expand to other cloud providers, I could see the need to move to Terraform, but for now, I feel confident in the choice as we've been able to achieve a lot of automation in a very short period of time and AWS manages everything for us. We have a small team and we've been conscience not to add to our tech debt as we develop our capabilities. In the spirit of openness and collaboration, and since nothing what I described is proprietary, I hope we are able to [open-source](https://github.com/thehartford) our implementation for others to follow if they so chose.

If you're interested in joining us to help build and design for the cloud, please take a look our [open positions](https://thehartford.taleo.net/careersection/2/jobdetail.ftl?job=2000911&tz=GMT-04%3A00&tzname=America%2FNew_York).

> [Now Go Build](https://aws.amazon.com/startups/NowGoBuild/) - Dr. Werner Vogels