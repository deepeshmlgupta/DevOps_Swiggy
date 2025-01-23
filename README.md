# 🚀 **DevOps Real-time Project: Swiggy Clone**

This project is a Swiggy clone 🍔 built with DevOps practices for scalability and security. It leverages AWS for cloud hosting and Terraform for managing infrastructure. Jenkins automates the CI/CD pipeline, while SonarQube ensures code quality. Security is prioritized using tools like OWASP and Trivy, and Docker is used to containerize the application for easier deployment and scaling.

#
## Project Overview
![Swiggy Project](https://github.com/user-attachments/assets/d078b6f3-a290-4584-9f2f-692ffe262ade)
#

## 🛠️ Tools & Services Used:

1. **Terraform** ![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)
2. **GitHub** ![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)
3. **Jenkins** ![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)
4. **SonarQube** ![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=flat-square&logo=sonarqube&logoColor=white)
5. **OWASP** ![OWASP](https://img.shields.io/badge/OWASP-000000?style=flat-square&logo=owasp&logoColor=white)
6. **Trivy** ![Trivy](https://img.shields.io/badge/Trivy-00979D?style=flat-square&logo=trivy&logoColor=white)
7. **Docker & DockerHub** ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white) ![DockerHub](https://img.shields.io/badge/DockerHub-2496ED?style=flat-square&logo=docker&logoColor=white)


## Cloning Project
```bash
git clone https://github.com/deepeshmlgupta/DevOps_Swiggy.git
```

## Installation

Terraform Installation

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform
```

AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```


Connect Your AWS to Virtual Machine
Now, connect you VM to you AWS account
```bash
aws configure
```   
```bash
AWS Access Key ID = <access key>
AWS Secret Access Key = <secret access key> 
```
![image](https://github.com/user-attachments/assets/0d08f4b7-c6fa-4641-a420-7adbe9f40b6e)


Now your AWS is connected with the VM but terrafrom will also connect the the AWS to run these command

```bash
export AWS_ACCESS_KEY_ID = <access key>
export AWS_SECRET_ACCESS_KEY = <secret access key> 
``` 


## Terraform Execution

```bash
cd Terraform
```

This command will initlize the terrafrom script in the Virtal Machine
```bash
terraform init
```
![image](https://github.com/user-attachments/assets/c7efbb12-d0f4-48ea-98b4-5e479aeab443)

This command will show you what all action your terraform script will perform on to your AWS Account 
```bash
terraform plan 
```
After Checking the plan of terraform if all thing gos well then apply it
```bash
terraform apply -auto-approve
```
![image](https://github.com/user-attachments/assets/5bbf6288-e0ff-4917-ab0f-19b7a6929c26)


## Docker
- <b>Let's first execute this project through Docker:</b>
  - Provide permission to docker socket so that docker build and push command do not fail
    ```bash
    chmod 777 /var/run/docker.sock
    ```
  - Now, build the Dockerfile
    ```bash
    docker build -t Swiggy:latest . --no-cache
    ```
  - After the image is build successfully now run the docker container
    ```bash
    docker run -d --name swiggy -p 3000:3000 deepeshmlgupta/swiggy:latest
    ```

> [!Note]
> Before running the jenkins pipeline you should make sure that you stop the current Docker Container

   - After the image is build successfully now run the docker container
    ```bash
    docker stop <container_ID>
    ```

## Jenkins Setup
- <b>Go to Jenkins Master and click on <mark> Manage Jenkins --> Plugins --> Available plugins</mark> install the below plugins:</b>
  - OWASP
  - SonarQube Scanner
  - Docker
  - Pipeline: Stage View
  - BlueOcean
#
- <b id="Owasp">Configure OWASP, move to <mark>Manage Jenkins --> Plugins --> Available plugins</mark> (Jenkins Worker)</b>
![image](https://github.com/user-attachments/assets/da6a26d3-f742-4ea8-86b7-107b1650a7c2)

- <b id="Sonar">After OWASP plugin is installed, Now move to <mark>Manage jenkins --> Tools</mark> (Jenkins Worker)</b>
![image](https://github.com/user-attachments/assets/3b8c3f20-202e-4864-b3b6-b48d7a604ee8)
#

## Sonar Setup
- <b>Login to SonarQube server and create the credentials for jenkins to integrate with SonarQube</b>
  - Navigate to <mark>Administration --> Security --> Users --> Token</mark>
  ![image](https://github.com/user-attachments/assets/86ad8284-5da6-4048-91fe-ac20c8e4514a)
  ![image](https://github.com/user-attachments/assets/6bc671a5-c122-45c0-b1f0-f29999bbf751)
  ![image](https://github.com/user-attachments/assets/e748643a-e037-4d4c-a9be-944995979c60)

#
- <b>Now, go to <mark> Manage Jenkins --> credentials</mark> and add Sonarqube credentials:</b>
![image](https://github.com/user-attachments/assets/0688e105-2170-4c3f-87a3-128c1a05a0b8)
#
- <b>Go to <mark> Manage Jenkins --> Tools</mark> and search for SonarQube Scanner installations:</b>
![image](https://github.com/user-attachments/assets/2fdc1e56-f78c-43d2-914a-104ec2c8ea86)

#
- <b>Go to <mark> Manage Jenkins --> System</mark> and search for SonarQube installations:</b>
![image](https://github.com/user-attachments/assets/ae866185-cb2b-4e83-825b-a125ec97243a)

#

## Weebhooks
-Sonar Webhook
  - <b>Login to SonarQube server, go to <mark>Administration --> Webhook</mark> and click on create </b>
  ![image](https://github.com/user-attachments/assets/16527e72-6691-4fdf-a8d2-83dd27a085cb)
  ![image](https://github.com/user-attachments/assets/a8b45948-766a-49a4-b779-91ac3ce0443c)
- Github Webhook
  - <b>Go to your github Repo, <mark>Setting --> Webhook</mark> and click on create </b>
  ![image](https://github.com/user-attachments/assets/c4d7c593-ac74-49af-a2ca-7eda4c2add6c)
  ![image](https://github.com/user-attachments/assets/fe38edc5-912d-4357-804f-872e684417cd)


#

## Jenkins Pipeline

- <b>Create a pipeline <mark>Swiggy-CICD</mark></b>
![image](https://github.com/user-attachments/assets/efea35bf-e662-499e-b36b-d79fb7f82580)
![image](https://github.com/user-attachments/assets/37f56ffd-07b1-4b09-a160-270725ed39ee)
- <b>Now Click, <mark>Apply --> Save</mark> and click on <mark>Build Now</mark> </b>
![image](https://github.com/user-attachments/assets/24c02437-2729-4d37-97d8-d95cdeaed632)




## About Me  
<img src="https://media.licdn.com/dms/image/v2/D5603AQGOyuk6Tn6-XA/profile-displayphoto-shrink_400_400/profile-displayphoto-shrink_400_400/0/1719034834385?e=1741824000&v=beta&t=CrlkrEiQGICb_GQIXvNw_CEG8bcifMm96JB4jiqYyQ0" alt="Deepesh Profile Image" width="150" height="150" style="border-radius:50%;">

**Deepesh Gupta**    
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/deepeshmlgupta/)  
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/deepeshmlgupta)  

---


