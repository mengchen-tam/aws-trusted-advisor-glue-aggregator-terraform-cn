# Description in Chinese
本方案主要由2个模块，下文简称admin模块和member模块。这两个模块会分开部署。
本方案中最重要的元素是两个部署在**admin模块里的lambda**，一个是**fetch-accounts-metadata**用于获取账号并触发SQS的Lambda，这里account id目前是hard code在脚本里的。客户可以提前修改或者部署以后修改。
这个lambda有2种触发方式：
1. terrafrom部署的scheduler, Amazon EventBridge > Rules > xxx-refresh-data-required-event
   默认是3天触发一次。可以在aws-trusted-advisor-glue-aggregator-terraform/main.tf里修改参数：
```data_ingestion_schedule_expression：
module "reporting-admin-standalone" {
  source                             = "./modules/reporting-admin-module"
  ...
  data_ingestion_schedule_expression = "rate(3 days)"
  ...
}
```
2. manual test. 直接在控制台里点击Test, lambda就会将accounts发送到SQS中。

其次就是**fetch-trusted-advisor**这个lambda，用于获取Member accounts report的lambda，会被SQS触发。这个lambda会使用admin_role去assume member_role, 然后使用trusted-advisor API 拿回查询结果。

Report获取完成之后会将结果写到S3桶里，同时也会创建一个glue table和Athena query，按照以下部署步骤完成以后即可看到结果。

# Deployment guide in Chinese
1.	下载并切换branch
```
Git clone https://github.com/mengchen-tam/aws-trusted-advisor-glue-aggregator-terraform-cn 
Git checkout cn-region-branch
```
2.	前置条件：准备好admin account和member account的credential profile。
```
[admin]
aws_access_key_id = <ak>
aws_secret_access_key = <sk>
[member]
aws_access_key_id = <ak>
aws_secret_access_key = <sk>
```
3.	开始部署admin模块中的资源
```
terraform apply -target=module.reporting-admin-standalone --profile=admin
```

4. 部署member模块: 这一步只是用admin_role_arn作为principle去在member account里部署一个role。
```
terraform apply -target=module.reporting-member-standalone -var profile=member
```
这一步也可以用其他的组织管理统一部署的role去替代，需要增加admin role的principle和以下policy:
```
"support:DescribeTrustedAdvisorChecks",
"support:DescribeTrustedAdvisorCheckResult"
```

5.	在admin account里修改lambda: xxx-fetch-accounts-metadata里面的accounts。也可在部署前修改modules/reporting-admin-module/src/lambda/functions/fetch_trusted_advisor/fetch_trusted_advisor.py  

```def process_event(self, event, context):

        accounts = []

        # TODO: add logic to fetch dynamicaly  the list of accounts in scope ...
        # BTW, here is also good place to save these accounts organisational metadata (owner, cost center, contact, etc.) into the data S3 bucket (to have even more data to analyse and join in the queries)
        # for this educative article we will just hardcode few accounts
        accounts.append('123456789012')
        accounts.append('111222333444')
```
这里可以根据自己情况，将account list贴入，也可以用其他方式，比如使用organizations list-accounts API去获取。
6. 手工触发lambda: xxx-fetch-accounts-metadata, 点击test
7. 最后就可以在S3 或者Athena里面去查询结果，或者集成athena到BI工具上用于展示。
8.  目前这个结果只有一个result字段，将所有属于某个account某个check-id的检查结果保存进去。如需和BI集成还需要做一些查询或者ETL的工作。

# aws-trusted-advisor-glue-aggregator

Code to deploy a solution to:
- periodically aggregate the Trusted Advisor results from different accounts to a centralised account, using AWS Lambda, AWS IAM, Amazon S3 and Amazon SQS 
- run analysis or report SQL queries on the aggregated data, using Amazon Athena, AWS Glue, Amazon S3

 ## Table of Contents

- [Description in Chinese](#description-in-chinese)
- [Deployment guide in Chinese](#deployment-guide-in-chinese)
- [aws-trusted-advisor-glue-aggregator](#aws-trusted-advisor-glue-aggregator)
  - [Table of Contents](#table-of-contents)
  - [Description](#description)
    - [Admin account module](#admin-account-module)
    - [Member account module](#member-account-module)
    - [Architecture](#architecture)
  - [Prerequisites](#prerequisites)
  - [Dependencies](#dependencies)
  - [Use](#use)
    - [Deployment](#deployment)
    - [Testing](#testing)
    - [Cleanup](#cleanup)
  - [Security](#security)
  - [License](#license)

## Description

Following the AWS Organization Unit naming convention, we refer to the central data-analysis account as the admin account; and to the target data accounts as the member accounts.

### Admin account module
This Terraform module creates resources supporting three flows:
- An Amazon EventBridge scheduler periodically invoke an AWS Lambda function to obtain the latest list of AWS Accounts to aggregate. It split the account ids into Amazon SQS Queue messages.
- AWS Lambda function is trigger for each account id in the Amazon SQS Queue, and chain roles to retrieve member account AWS Trusted Advisor checks results. It write them in raw JSON format in a centralised Amazon S3 bucket.
- AWS  Glue map raw-data structure to a synthetic RDMS data-model that can be query in SQL via Amazon Athena, to generate CSV reports or extracts in an Amazon S3 bucket. 
The module includes all needed roles for the proper access of the services.
The module includes logs eviction management for the relevant services.

### Member account module

This Terraform module creates an AWS IAM role allowed to read AWS Trusted Advisor checks results and create trust for the admin account role to assume it.

### Architecture
The following diagram describes the full architecture.

![Diagram](.images/TrustedAdvisor.png)

**Blue flow:** obtains list of accounts in scope
1. Periodically trigger the "refresh data" process
2. Retrieve list of accounts. Created an SQS message for each account ID

**Yellow flow:** retrieve Trusted Advisor data 
1. Invoke Lambda function for each message 
2. Assume trusted admin role
3. Assume member account role
4. Call Trusted Advisor API to get data
5. Save data in S3 Bucket

**Green flow:** analyse data
1. Run an Athena Query
2. Athena Query look how to map the raw data to the synthetic data-model
3. Athena Query read and process the data
4. Athena Query save the query result


## Prerequisites
 
* **AWS Premium Support subscription**: 
AWS Business Support or AWS Enterprise Support subscription is required to use this code, as it leverage AWS Trusted Advisor APIs which are available only to these levels of subscription.

## Dependencies

* **terraform**: 1.7.5 [Reference](https://github.com/hashicorp/terraform)

## Use

The available variables are described in [variables.tf](./variables.tf) file for each module.

### Deployment

> Pay attention:
Both modules are meant to be used as standalone modules. They have to be deployed independently to the relevant AWS accounts
The Member module is to be deployed on each member account. 

**Option 1:**
You can inspire from [main.tf](./main.tf) to use the modules directly within your code.    
Please have a look inside inside [variables.tf](./variables.tf) for all the possible options.

**Option 2:**
Alternatively, if you have [Terraform](https://www.terraform.io/) installed on your workstation, you can deploy the example by executing:

```bash
export AWS_PROFILE=<profile>
export AWS_DEFAULT_REGION=cn-north-1

terraform plan -target=module.reporting-admin-module -var region=$AWS_DEFAULT_REGION -var profile=$AWS_PROFILE
terraform apply -target=module.reporting-admin-module -var region=$AWS_DEFAULT_REGION -var profile=$AWS_PROFILE

terraform plan -target=module.reporting-member-module -var region=$AWS_DEFAULT_REGION -var profile=$AWS_PROFILE
terraform apply -target=module.reporting-member-module -var region=$AWS_DEFAULT_REGION -var profile=$AWS_PROFILE
```

> Pay attention:
you should first modify the `AWS_DEFAULT_REGION` in accordance to your requirements.

### Testing

Each organisation has its own way to maintain and expose its inventory of AWS Accounts.
It is beyond the scope of this article to cover all the options to choose as scope of member accounts (ie: static list, database/file dynamic list, AWS Organization Unit based, etc.)

To support accounts dynamically joining and exiting the scope of analysis, the list of member accounts is re-evaluated each time at runtime.   
This educational code allow to hardcode simple list of 2-3 accounts in [fetch_accounts_metadata.py](./modules/reporting-admin-module/src/lambda/functions/fetch_accounts_metadata/fetch_accounts_metadata.py) for immediate testing purpose. 
But the reader should replace it by custom logic, adapted to its organisation context, for more advanced usage.

**Option 1: AWS Console**
You can use the AWS Console to:
- see raw data files aggregated in Amazon S3 bucket
- run Amazon Athena named or custom queries
- see query results in Amazon S3 bucket

**Option 2: AWS CLI**

```bash
export randomPrefix=<prefix output displayed by Terraform at deployment>

aws lambda invoke --function-name $randomPrefix-reporting-fetch-accounts-metadata:LIVE out.json
jq --color-output . out.json

aws logs tail /aws/lambda/$randomPrefix-reporting-fetch-accounts-metadata
aws logs tail /aws/lambda/$randomPrefix-reporting-fetch-trusted-advisor
aws s3 ls s3://$randomPrefix-reporting
```

### Cleanup

Use with caution:

```bash
rm out.json
terraform destroy -var region=$AWS_DEFAULT_REGION -var profile=$AWS_PROFILE
```

## Security

See [CONTRIBUTING](CONTRIBUTING.md) for more information.

## License

This project is licensed under the MIT-0 License.

=======

