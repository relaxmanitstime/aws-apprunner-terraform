# AWS CodePipeline, Amazon RDS, Terraform을 사용하여 AWS App Runner에 Spring PetClinic 애플리케이션 빌드 및 배포 

![Build Status](https://codebuild.us-east-1.amazonaws.com/badges?uuid=eyJlbmNyeXB0ZWREYXRhIjoiSy9rWmVENzRDbXBoVlhYaHBsNks4OGJDRXFtV1IySmhCVjJoaytDU2dtVWhhVys3NS9Odk5DbC9lR2JUTkRvSWlHSXZrNVhYQ3ZsaUJFY3o4OERQY1pnPSIsIml2UGFyYW1ldGVyU3BlYyI6IlB3ODEyRW9KdU0yaEp6NDkiLCJtYXRlcmlhbFNldFNlcmlhbCI6MX0%3D&branch=master)
[![Gitpod Ready-to-Code](https://img.shields.io/badge/Gitpod-ready--to--code-blue?logo=gitpod)](https://gitpod.io/#https://github.com/aws/aws-cdk)
[![NPM version](https://badge.fury.io/js/aws-cdk.svg)](https://badge.fury.io/js/aws-cdk)
[![PyPI version](https://badge.fury.io/py/aws-cdk.core.svg)](https://badge.fury.io/py/aws-cdk.core)
[![NuGet version](https://badge.fury.io/nu/Amazon.CDK.svg)](https://badge.fury.io/nu/Amazon.CDK)

## 개요

![AwsAppRunner](images/aws-app-runner-description.png)

AWS App Runner, Amazon RDS, AWS CodePipeline, AWS CodeCommit, AWS CodeBuild 등 AWS CI/CD 툴을 사용하여 엔지니어가 서버리스 컨테이너 워크로드를 위한 파이프라인을 구축할 수 있도록 한다.   
   
[AWS App Runner](https://aws.amazon.com/apprunner/) 는 컨테이너형 웹 애플리케이션을 규모에 맞게 배포 및 실행하기 위한 AWS 모범 사례와 기술을 활용한다.   
이를 통해 새로운 애플리케이션 및 기능을 출시하는데 소요되는 시간을 대폭 단축된다.   
   
App Runner는 AWS ECS와 Fargate 위에서 실행되고, 휠씬 더 쉬운 접근성과 비용 추정이 간단하다.([초당 고정 CPU/메모리 요금 부과](https://aws.amazon.com/ko/apprunner/pricing/))   

## 배경

스프링 프레임워크는 다양한 유형의 산업 애플리케이션을 구축하기 위해 독립적/집합적으로 사용될 수 있는데 초점을 맞추고 느슨하게 결합된 작은 자바 프레임워크의 집합이다.   
   
스프링 펫클리닉 샘플 애플리케이션은 스프링 애플리케이션 프레임워크를 사용하여 단순하지만 강력한 데이터베이스 지향 애플리케이션을 구축하는 방법을 보여주기 위해 설계되었다.   
   
백엔드는 AWS RDS(MySQL)를 사용하며 스프링의 핵심 기능 활용을 시연한다.   

## 기여자

1. Irshad A Buchh, Amazon Web Services
2. relaxmanitstime

## 아키텍처
![Architecture](images/Architecture.png)

## 사전 준비 사항

### AWS account

Administrator 권한이 있는 AWS 계정   
(참고. 운영 환경에서는 파이프라인 운영에 필요한 최소한의 권한으로 해야 함)   

### AWS Cloud9 환경 생성

AWS 관리 콘솔에 로그인 후 Cloud9 서비스를 검색 후 실행하여 ap-northeast-1 도쿄 리전에서 Amazon Linux2 기반으로 환경을 구성한다. 여기서는 t3.medium으로 선택한다. 참고로 AWS App Runner는 22-08-05 기준으로 서울 리전에서는 사용할 수 없다.   

![AwsCloud9Preview](images/aws-cloud9-01.png)

> Step 1. Name environment : 이름은 적당히...   
> Step 2. Configure settings : Create a new EC2 instance for ...(direct access), t3.medium, Amazon Linux2 나머지는 Default 설정 그대로...   

### AWS Cloud9 환경 구성

AWS Cloud9 IDE를 실행 후 Welcome 탭을 닫고 새 터미널 탭을 연다.   

![AwsCloud9](images/aws-cloud9-220805.png)

#### Cloud9 인스턴스에 대한 IAM 역할 생성 및 연결

기본적으로, Cloud9은 임시 IAM 자격 증명을 관리하나, 테라폼과 호환되지 않는다.   
Cloud9 임시 자격 증명을 사용하지 않도록 설정하고 Cloud9 인스턴스에 대한 IAM 역할을 생성 후 연결해야 한다.   

1. [관리자 권한을 가진 IAM Role 생성](https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess)
1. **AWS service**, **EC2** 선택 후 권한 확인을 위해 **Next** 클릭
1. **AdministratorAccess** 선택되어 있는지 확인 후, 태그를 할당하기 위해 **Next: Tags** 클릭
1. 기본값을 사용하고, **Next: Review** 클릭
1. Name에 **workshop-admin** 입력후 **Create role** 클릭
![createrole](images/createrole.png)
1. [Cloud9 EC2 인스턴스 링크](https://console.aws.amazon.com/ec2/v2/home?#Instances:tag:Name=aws-cloud9-;sort=desc:launchTime)
1. 인스턴스 선택 후, 마우스 우측 버튼 **보안 / IAM 역할 수정** 선택. 참고: 메뉴 옵션이 없다면, **Actions / Instance Settings / Modify IAM Role** 을 대신 선택.
![c9instancerole](images/c9instancerole.png)
1. IAM 역할에 **workshop-admin** 드롭다운 선택 후 **Save** 선택
![c9attachrole](images/c9attachrole.png)
1. Cloud9 워크스페이스로 돌아가 우측 상단 기어 아이콘 클릭하여 환경설정 열기(Open Preferences)를 선택
1. **AWS SETTINGS** 선택
1. **AWS managed temporary credentials** 옵션을 끈다
1. 탭 닫기
![c9disableiam](images/c9disableiam.png)
1. Cloud9 터미널에서 아래 명령어 실행
    ```bash
    rm -vf ${HOME}/.aws/credentials
    ```
1. 마지막으로, [GetCallerIdentity](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html) CLI 명령어를 사용해 Cloud9 IDE가 올바른 IAM 역할을 사용하고 있는지 확인.
    ```bash
    aws sts get-caller-identity --query Arn | grep workshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
    ```
    ![AwsGetCallerIdentity](images/aws-get-caller-identity-result.png)

#### awscli 설정

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

설치 후 탭을 새로 연다.   

```bash
aws --version
pip install awscli --upgrade --user
```

aws configure를 통해 아래처럼 설정한다.      

```
admin:~/environment $ aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: ap-northeast-1
Default output format [None]: 
```

#### 테라폼 설치

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

terraform version
```

#### Apache Maven 설치 및 설정

아래와 같이 설치한다.   

```bash
cd /tmp
sudo wget https://archive.apache.org/dist/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
sudo tar xf /tmp/apache-maven-*.tar.gz -C /opt
sudo ln -s /opt/apache-maven-3.8.1 /opt/maven
```

아래와 같이 bash 설정으로 들어간 후,   

```bash
sudo nano ~/.bashrc
```

파일 끝에 다음을 붙여넣고 저장 후 빠져나온다.   

```bash
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=${M2_HOME}/bin:${PATH}
```

아래와 같이 설치되었음을 확인한다.   

```bash
source ~/.bashrc
mvn --version
```

#### 리포지토리 클론
<!---
You will need to import the workshop files into your Cloud9 environment:

```bash
wget https://github.com/aws-samples/aws-ecs-cicd-terraform/archive/master.zip
unzip master.zip
cd aws-ecs-cicd-terraform-master
```
-->

소스 코드 리포지토리를 클론한다.   

```bash
cd ~/environment
git clone https://github.com/relaxmanitstime/aws-apprunner-terraform.git
```

## Apache Maven을 이용하여 응용 프로그램 패키징

아래 명령을 처음 실행할 때 메이븐은 필요한 플러그인과 관련 종속성을 다운로드하는 데 시간이 걸린다.(5분 이상)   
이후에는 다운로드할 필요가 없어서 더 빨리 실행한다.   
컴파일된 자바 클래스는 spring-petclinic/target/classes에 배치된다.   

```bash
cd ~/environment/aws-apprunner-terraform/petclinic
mvn package -Dmaven.test.skip=true
```

## Petclinic 도커 이미지 빌드 및 태그 지정

```bash
docker build -t petclinic .
```

## Petclinic 애플리케이션을 로컬로 실행

```bash
docker run -it --rm -p 8080:80 --name petclinic petclinic
```

컨테이너 80 포트를 사용하여 애플리케이션이 실행되고, 호스트 8080 포트에 노출된다.   
상단 메뉴 `Preview` 메뉴의 `Preview Running Application`을 클릭하면, 스프링 펫클리닉 애플리케이션을 표시하는 브라우저가 열린다.   

![AwsCloud9MenuPreview](images/aws-cloud9-menu-preview.png)

![DockerLocalRun220805](images/docker-local-run-220805.png)

## Petclinic 도커 이미지를 Amazon ECR에 푸시
새 터미널을 열고, 다음을 실행한다.   

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
AWS_REGION=$(aws configure get region)

export REPOSITORY_NAME=petclinic
export IMAGE_NAME=petclinic

aws ecr create-repository \
    --repository-name $REPOSITORY_NAME \
    --image-scanning-configuration scanOnPush=true \
    --region $AWS_REGION
	
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

docker tag $IMAGE_NAME $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_NAME
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_NAME
```

그 후, AWS ECR의 리포지토리 메뉴로 가서 petclinic을 삭제한다.(이후 테라폼에서 에러 발생 방지 목적)   

![EcrRepositoriesPetclinic](images/amazon-elastic-container-registry-repositories-petclinic.png)

## 인프라 및 파이프라인 구축

테라폼을 사용해 AWS Code Pipeline을 포함한 아키텍처를 구축한다.(유료 리소스를 생성하므로 비용절감을 위해 향후 리소스를 정리하는 것이 좋다.)   

### DB 암호에 대한 SSM 매개변수 설정

```bash
aws ssm put-parameter --name /database/password --value mysqlpassword --type SecureString
```

### 테라폼 변수 편집

```bash
cd ~/environment/aws-apprunner-terraform/terraform
```

![AwsApprunnerTerraformTerraformTfvars](images/aws-apprunner-terraform-terraform-tfvars.png)

`terraform.tfvars` 파일을 수정한다.   
- aws_profile은 "default" 그대로 둔다.   
- aws_region은 `"ap-northeast-1"`으로 수정하고,   
- codebuild_cache_bucket_name의 apprunner-cache-yyyymmdd-identifier 중 가운데 `yyyymmdd`는 오늘 날짜로, `identifier`는 사용자 고유한 이름으로 수정한다.   

S3 버킷의 이름은 전역적으로 고유해야 하고, 숫자, 소문자, 하이픈이 포함될 수 있다.   

### 빌드

테라폼을 초기화한다.   

```bash
terraform init
```

테라폼을 사용하여 인프라 및 파이프라인을 구축한다.   

```bash
terraform apply
```

깃 이메일과 사용자 계정 입력 후 테라폼 실행 계획이 표시된다.   
작업을 계속할지 묻는 메세지가 나오면 `yes`를 입력한다.   

“terraform apply”을 완료하는 데 몇 분 정도 걸린다.   

![TerraformApplyResult](images/terraform_apply_result.png)

### 구축한 스택 탐색

빌드가 완료되면 AWS 콘솔을 사용하여 환경을 탐색할 수 있다.   
- App Runner 서비스 [AWS App Runner console](https://console.aws.amazon.com/apprunner/)
- RDS DB [Amazon RDS console](https://console.aws.amazon.com/rds).
- ECR repo [Amazon ECR console](https://console.aws.amazon.com/ecr).
- 코드커밋 리포 [AWS CodeCommit console](https://console.aws.amazon.com/codecommit).
- 코드빌드 프로젝트 [AWS CodeBuild console](https://console.aws.amazon.com/codebuild).
- 코드 파이프라인 [AWS CodePipeline console](https://console.aws.amazon.com/codepipeline).

코드커밋 리포에 빌드한 코드가 없기 때문에 파이프라인이 실패한 상태에서 시작된다.   
다음 단계에서는 파이프라인을 트리거하기 위해 펫클리닉 앱을 repo에 푸시한다.    

### App Runner 서비스 탐색
App Runner 서비스 설정파일 [terraform/services.tf](terraform/services.tf)을 열어서 옵션들을 확인한다.   

```typescript
 image_repository {
      image_configuration {
        port = var.container_port
        runtime_environment_variables = {
          "spring.datasource.username" : "${var.db_user}",
          "spring.datasource.password" : "${data.aws_ssm_parameter.dbpassword.value}",
          "spring.datasource.initialization-mode" : var.db_initialize_mode,
          "spring.profiles.active" : var.db_profile,
          "spring.datasource.url" : "jdbc:mysql://${aws_db_instance.db.address}/${var.db_name}"
        }
      }
      image_identifier      = "${data.aws_ecr_repository.image_repo.repository_url}:latest"
      image_repository_type = "ECR"
    }
```
**참고:** 운영 환경에서는 latest 태그 대신 의미 있는 태그를 사용하는 것이 좋다.   

## 파이프라인을 사용하여 펫클리닉 애플리케이션 배포

깃으로 푸시하여 파이프라인을 통해 펫클리닉 애플리케이션을 배포한다.    

### 펫클리닉 애플리케이션을 위한 로컬 깃 리포 설정

```bash
cd ~/environment/aws-apprunner-terraform/petclinic
```

깃 사용자 이름과 이메일 주소를 설정한다.   

```bash
git config --global user.name "YourName"
git config --global user.email "YourEmail@example.com"
```

다음과 같이 펫클리닉을 위한 로컬 깃 리포를 만든다.   

```bash
git init
git add .
git commit -m "Baseline commit"
```

### 리모트 코드커밋 리포를 설정

너가 만든 파이프라인의 일부로 AWS CodeCommit Repo가 만들어졌다.   
이제 로컬 펫클리닉 리포에 대한 리모트 리포를 설정할 것이다.   

인증을 위해 AWS IAM git credential helper를 사용해 IAM 역할 사용 권한에 따라 Git 자격 증명을 생성할 수 있다.   

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

테라폼 빌드의 아웃풋으로부터 다음 단계에서 사용할 `source_repo_clone_url_http`을 사용한다.   

```bash
cd ~/environment/aws-apprunner-terraform/terraform
export tf_source_repo_clone_url_http=$(terraform output --raw source_repo_clone_url_http)
```

깃 리포를 위한 리모트를 설정한다.   

```bash
cd ~/environment/aws-apprunner-terraform/petclinic
git remote add origin $tf_source_repo_clone_url_http
git remote -v
```

아래와 같은 결과가 나와야 한다.   

```bash
origin  https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/petclinic (fetch)
origin  https://git-codecommit.ap-northeast-1.amazonaws.com/v1/repos/petclinic (push)
```

### 파이프라인 실행(트리거)

파이프라인을 실행하기 위해서 리모트에 마스터 브랜치를 푸시한다.   

```bash
git push -u origin master
```

파이프라인은 코드를 pull하고, 도커 이미지를 빌드 및 ECR에 푸시하고, ECS 클러스터에 배포한다.   
이 작업은 30분 이상(?) 걸린다.   
[AWS CodePipeline console](https://console.aws.amazon.com/codepipeline) 에서 파이프라인을 모니터링할 수 있다.   
![AwsCodePipelineConsole](images/aws-codepipeline-console.png)

### 애플리케이션 테스트

```bash
cd ~/environment/aws-apprunner-terraform/terraform
export tf_apprunner_service_url=$(terraform output apprunner_service_url)
echo $tf_apprunner_service_url
```

테라폼 빌드 아웃풋을 통해 `apprunner_service_url` URL을 브라우저를 통해 액세스한다.   

![Petclinic](images/petclinic.png)

## 변경사항을 파이프라인에 적용하고 재테스트

이제 파이프라인을 사용하여 변경사항을 애플리케이션에 배포할 수 있다.   
아래와 같이 환영 메세지를 변경하여 재테스트할 수 있다.

```
cd ~/environment/aws-apprunner-terraform/petclinic
vi src/main/resources/messages/messages.properties
```

웰컴 문자열을 예를 들어 "Hello"로 변경 후 변경사항을 커밋한다.   

```
git add .
git commit -m "Changed welcome string"
```

파이프라인을 트리거하기 위해 변경사항을 푸시한다.   

```bash
git push origin master
```

이전과 마찬가지로, 콘솔을 사용하여 파이프라인을 통한 변경 진행 사항을 확인할 수 있다.   
작업이 완료되면, 애플리케이션이 수정된 환영 메세지로 수정되었는지 확인한다.   

## 스택 해체 및 리소스 삭제

**참고:**
이벤트 엔진과 제공된 AWS 계정을 사용하여 AWS가 호스팅하는 이벤트에 참여하는 경우 이 단계를 완료할 필요가 없으나,(관리자에 의해 향후 정리)   
불필요한 과금을 피하기 위해서는 아래와 같이 스택을 해체해서 리소스를 없애야 한다.   

아래 명령어를 통해 Amazon ECR를 삭제한다.   

```bash
aws ecr delete-repository \
    --repository-name $REPOSITORY_NAME \
    --region $AWS_REGION \
    --force
```

아래 명령어를 입력 후 `yes`를 입력해 스택을 종료한다.   
```
cd ~/environment/aws-apprunner-terraform/terraform
terraform destroy
```

완료되면 파이프라인에서 사용하는 S3 버킷을 수동으로 비우고 삭제해야 한다.   
