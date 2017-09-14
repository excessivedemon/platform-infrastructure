# platform-infrastructure

## Overview
This project automates the creation of all necessary AWS resources for deploying a CI/CD pipeline for the [platform-application](https://github.com/excessivedemon/platform-application) project. Upon successful deploy in AWS, any succeeding code changes on the `staging` or `master` branches of the [platform-application](https://github.com/excessivedemon/platform-application) project trigger [AWS Codepipeline](https://aws.amazon.com/codepipeline/) to build and test *(using [AWS Codebuild](https://aws.amazon.com/codebuild/))*, and deploy *(on  [AWS Lambda](https://aws.amazon.com/lambda/) and [Amazon API Gateway](https://aws.amazon.com/api-gateway/))* a new release.

## Prerequisites
- [Python Virtual Environment](http://python-guide-pt-br.readthedocs.io/en/latest/dev/virtualenvs/)
- [AWS Credentials](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)

## Getting Started

Clone this project

	$ cd ~/Projects
	$ git clone https://github.com/excessivedemon/platform-infrastructure.git
	$ cd platform-infrastructure

Create your virtual environment

	$ virtualenv ~/platform-infrastructure

Activate your virtual environment

	$ source ~/platform-infrastructure/bin/activate

Install required packages

	$ pip install -r requirements.txt
	
Copy the existing `config.yaml.template` reference file into `config.yaml` and modify to match your configuration (see section on *Configuration*).

	$ cp config.yaml{.template,}

### Configuration
The `config.yaml` configuration file comprises a `deployment` section for all AWS resource deployment related configuration and a `github` section for all github related configuration.

#### `deployment`

| key | description |
| --- | --- |
| project_name | prefix used for deploying AWS resources |
| region | AWS region to deploy cloudformation stacks to |
| profile | AWS profile to use when deploying AWS resources; see [Named Profiles](http://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html) |
| role_name | AWS IAM role to assume in another AWS account; only used when authenticating in a separate AWS *(identity)* account and deploying resources in another AWS account |
| account_id | AWS account *(id)* where the value for role_name *(above)* exists; only used when authenticating in a separate AWS *(identity)* account and deploying resources in another AWS account |
| audit_tags | AWS resource tags applied to deployed AWS resources (for auditing and cost allocation purposes); see [AWS Tagging Strategies](https://aws.amazon.com/answers/account-management/aws-tagging-strategies/) |

#### `github`

| key | description |
| --- | --- |
| Owner | GitHub username |
| Repo | GitHub repository name containing the application to be deployed; typically the repository where you cloned [platform-application](https://github.com/excessivedemon/platform-application) |
| OAuthToken | GitHub OAuthToken for granting AWS CodePipeline access to the GitHub repo; see [Create a personal access token for the command line](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) |
| staging_branch | git branch for staging environment |
| production_branch | git branch for production environment |
| repo_url | GitHub repository url containing the application to be deployed; typically the repository url where you cloned [platform-application](https://github.com/excessivedemon/platform-application) |

### Deploy

Create the stack

	$ ansible-playbook main.yaml -e @config.yaml

### API Invoke URL
The API Invoke URL for both `staging` and `production` stages are accessible from the outputs of a successful deploy *(above)* command. An example of such *(truncated)* output looks like:

```
...
TASK [fetch staging api invoke url] ****************************************************
changed: [localhost]

TASK [debug] ***************************************************************************
ok: [localhost] => {
    "msg": "Staging API invoke URL is https://71ooo9scjj.execute-api.ap-southeast-2.amazonaws.com/staging"
}

TASK [fetch production api invoke url] *************************************************
changed: [localhost]

TASK [debug] ***************************************************************************
ok: [localhost] => {
    "msg": "Production API invoke URL is https://71ooo9scjj.execute-api.ap-southeast-2.amazonaws.com/production"
}
...
```
You can prepend the endpoints outlined at [platform-application](https://github.com/excessivedemon/platform-application) with the API invoke urls like so:

#### Production
Sample production API call on the `health` endpoint:

```
$ curl https://71ooo9scjj.execute-api.ap-southeast-2.amazonaws.com/production/health
{"status_code": 200, "response_time": 1.02}
```
#### Staging
Sample staging API call on the `metadata` endpoint:

```
$ curl https://71ooo9scjj.execute-api.ap-southeast-2.amazonaws.com/staging/metadata
{"lastcommitsha": "0b2d45f16549aaecd44ba12519c930c449bb8c64", "version": "0.1"}
```

## Architecture

All infrastructure codes are standard [AWS CloudFormation](https://aws.amazon.com/cloudformation/) codes, written as [Jinja2](http://jinja.pocoo.org/) templates to support code reuse, parameter substitution, and more advanced control structures and flow not supported by plain [AWS CloudFormation](https://aws.amazon.com/cloudformation/).

All automation and orchestration are implemented in [Ansible](https://www.ansible.com/) as [Ansible Playbooks](http://docs.ansible.com/ansible/latest/playbooks.html)

For the most part, deploying the CI/CD pipeline involves transpiling [Jinja2](http://jinja.pocoo.org/) templates into [AWS CloudFormation](https://aws.amazon.com/cloudformation/) templates and then creating [AWS CloudFormation](https://aws.amazon.com/cloudformation/) stacks using the ansible [cloudformation module](http://docs.ansible.com/ansible/latest/cloudformation_module.html).

### Infrastructure


#### [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
A single API Gateway is configured with a `/{proxy+} ANY` integration with an [AWS Lambda](https://aws.amazon.com/lambda/) function. The integration is configured to call a specific lambda function alias *(LambdaFunctionAlias)* based on the configured stage variable. The `staging` stage has a `LambdaFunctionAlias` value of `staging` and the `production` stage has a `LambdaFunctionAlias` value of `production`. This is what allows API gateway to serve different versions of the same [AWS Lambda](https://aws.amazon.com/lambda/) function, depending on what stage is being invoked.

#### [AWS Lambda](https://aws.amazon.com/lambda/)
The [platform-application](https://github.com/excessivedemon/platform-application) project is deployed as a single lambda function, with two aliases: `staging` and `production`. Each alias can point to any lambda version.

#### [AWS Codepipeline](https://aws.amazon.com/codepipeline/)
There are two pipelines - one for the `staging` environment and one for the `production` environment. The `staging` pipeline watches for code changes on the `staging` branch while the `production` pipeline watches for code changes on the `master` branch.

#### [AWS Codebuild](https://aws.amazon.com/codebuild/)
There are two codebuild projects - one for the `staging` environment and one for the `production` environment. A code change in the `staging` branch triggers the `staging` pipeline which in turn triggers the install phase, build *(and deploy)* phase, and the post_build phase on the `staging` codebuild project. The same is true for `production`.

#### Build and Deploy
The build spec is outlined on the [platform-application](https://github.com/excessivedemon/platform-application)'s [buildspec.yaml](https://github.com/excessivedemon/platform-application/blob/master/build/buildspec.yaml) file. The process involves installing [ansible](https://www.ansible.com/) on the install phase, calling the [deploy](https://github.com/excessivedemon/platform-application/blob/master/build/deploy.yaml) playbook on the build *(and deploy)* phase, and then running the application's [tests](https://github.com/excessivedemon/platform-application/blob/master/app_tests.py) on the post_build phase.

The [deploy.yaml](https://github.com/excessivedemon/platform-application/blob/master/build/deploy.yaml) playbook is responsible for creating the files required by the `metadata` and `health` endpoints, packaging the application, updating the lambda function, publishing a lambda version, and associating the corresponding lambda alias (`staging` if running the `staging` codebuild project, otherwise `production`) to the newly published lambda version.

In the event of a failure in any of the phases, the build fails and no code is deployed.

#### Rollback strategy
Flexibility to point lambda aliases (`staging` or `production`) to previous lambda versions, or simply undo a commit and the CI/CD pipeline will automatically build and deploy a new release.

#### Promotion from staging to production
Upon satisfactory completion of further tests (example: User Acceptance Tests), changes from `staging` branch can be merged to `master` branch, which will automatically trigger the CI/CD pipeline to build and deploy a new release.

#### Gotchas
+ There is currently no out of the box support for [GitHub](https://github.com/) PR integration with [AWS Codebuild](https://aws.amazon.com/codebuild/) build results
+ on initial deploy of this project, both `staging` and `production` lambda aliases will point to the `$LATEST` lambda alias (which will point to the latest commit in the `master` branch) - the aliases are correctly updated on first successfull build *(and deploy)*
+ `{"message":"Missing Authentication Token"}` error when no path is specified on the API invoke URL similar to the issue here: ([https://github.com/serverless/serverless/issues/1738](see https://github.com/serverless/serverless/issues/1738); Workaround is to call `/root` instead of `/`
