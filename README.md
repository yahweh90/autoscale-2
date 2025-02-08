1. Install and Run Jenkins Docker container

```sh
# documentation
# https://github.com/jenkinsci/docker/blob/master/README.md

docker run -d -v jenkins_home:/var/jenkins_home -p 8080:8080 -p 50000:50000 --restart=on-failure jenkins/jenkins:lts-jdk17
```

2. Open Jenkins Web UI
- [localhost: 8080]

3. Get Jenkins Admin Password
    - If containers are not running on detached mode the logs will show the Admin Password
    - If containers are running in detached mode

```sh
# access container terminal
docker exec -it $(docker ps -a | grep jenkins | awk '{print $1}') sh

# cat Jenkins initial password
cat /var/jenkins_home/secrets/initialAdminPassword
```

4. Enter password on Web UI
    1. Create username and password
    2. Install recommended plugins
    3. Install Git: `Manage Plugin` --> `Plugins` --> `Available Plugins` --> `Search for Git`
5. Create:
    1. AWS User: `jenkins`
    2. Create: `Secret_Key and Access_Key`
6. Clone the repo:

```sh
git clone [repo_name]

# Navigate to repo directory
cd [repo_name]

# Open folder with VS Code
code .
```

7. Create Jenkins Credentials
- Manage Jenkins --> Credentials --> System --> Global Credentials (unrestricted) --> Add Credentials
- Kind: `AWS Credentials`
- ID: `AWS_KEYS`
- Description: `AWS_SECRET_ACCESS_KEY`
- Access Key ID: `[Add_access_key_id`

8. Install Require Tools
    - awscli
    - terraform

```sh
# access container terminal
# install awscli and terraform

docker exec -u 0 -it $(docker ps -a | grep jenkins | awk '{print $1}') sh -c ' apt update && apt install -y awscli; mkdir -p /home/jenkins/bin; curl -fsSL https://releases.hashicorp.com/terraform/1.5.7/terraform_1.5.7_linux_amd64.zip -o /home/jenkins/terraform.zip; unzip /home/jenkins/terraform.zip -d /home/jenkins/bin; rm /home/jenkins/terraform.zip; export PATH="/home/jenkins/bin:$PATH"; mv /home/jenkins/bin/terraform /usr/local/bin; terraform --version'
```

Jenkins Setup

Through classic UI --> New Item
1. `Enter an item name`
2. `select Pipeline`
3. `Click OK`
4. `Click Pipeline`
5. `Scroll down to Pipeline section`
6. In the Pipeline section ensure the definition indicates the `Pipeline script from SCM` option
7. Choose the repo containing the Jenkinsfile
8. Specify the branch: `main` (for this lab)
9. Script Path: `Jenkinsfile`

Add Jenkinsfile in the root directory of the rep:

Jenkinsfile

```groovy
// Define the Jenkins pipeline using declarative syntax
pipeline {
    // run on any available agent/node
    agent any
    // define global environment variables
    environment {
        AWS_REGION = 'us-east-1' // set the AWS region for this deployment
        TERRAFORM_DIR = '.' // set the working directory for Terraform
    }

    // options for the whole pipeline
    options {
        // set maximum execution time, to prevent build taking too long or hanging
        // timeout after 1 hour
        timeout(time: 1, unit: 'HOURS')
        // prevent multiple builds from running at the same time
        disableConcurrentBuilds()
    }

    // define the pipeline stages (1 or more stages)
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs() // remove all files from workspace
            }
        }

        stage('Checkout code') {
            steps {
                // clone the main branch from the given directory
                git branch 'main',
                    url: 'https://github.com/SerginoDelia/autoScale/'
            }
        }

        // Initialize Terraform and Terraform plan (create execution plan)
        stage('Terraform Init and Plan') {
            steps {
                withAWS { // use custom AWS credentials wrapper
                    // terraform init - initialize terraform working dir
                    // terraform plan -out=tfplan - create and save execution plan
                    sh '''
                        terraform init
                        terraform plan -out=tfplan
                    '''
                }
            }
        }

        // manual approval step before applying changes
        stage('Approval') {
            // prompt user for manual approval before applying the terraform
            input message: "Do you want to apply the terraform plan?",
                ok: "Apply Plan"
        }

        stage('Terraform Apply') {
            steps {
                withAWS {
                    // apply terraform without asking for approval confirmation
                    sh 'terraform apply --auto-approve tfplan'
                }
            }
        }
    }

    // post-build actions
    post {
        // actions to run regardless of build result
        always {
            // remove/clean terraform plan file
            // can also create logic to save the tfplan-[build_number] based on build number - can add later or
            sh 'rm -f tfplan'
        }
        // on successful build run this action
        success {
            echo 'Infrastructure deployment completed successfully.'
        }
        // action for build failure
        failure {
            echo 'Infrastructure deployment failed'
        }
        // final cleanup
        cleanup {
            cleanWs() // clean workspace after build
        }
    }
}

// custom helper function/method for AWS credentials
def withAWS(Closure body) {
    // bind AWS credentials from Jenkins credentials
    withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: 'AWS_SECRET_ACCESS_KEY' // name of credentials created in Jenkins
    ]]) {
        // export AWS credentials as environment variables
        sh """
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_REGION=$AWS_REGION
        """
        body() // execute the passed closure with AWS credentials set
    }
}
```


