pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        TERRAFORM_DIR = '.'
    }
    
    options {
        // Add timeout and cleanup options
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout Code') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/derrickSh43/autoScale'
            }
        }
        
        stage('Terraform Init & Plan') {
            steps {
                withAWS {
                    sh '''
                        terraform init
                        terraform plan -out=tfplan
                    '''
                }
            }
        }
        
        stage('Approval') {
            steps {
                input message: "Do you want to apply the terraform plan?",
                      ok: "Apply Plan"
            }
        }
        
        stage('Terraform Apply') {
            steps {
                withAWS {
                    sh 'terraform apply -auto-approve tfplan'
                }
            }
        }
    }
    
    post {
        always {
            // Clean up terraform files
            sh 'rm -f tfplan'
        }
        success {
            echo 'Infrastructure deployment completed successfully!'
        }
        failure {
            echo 'Infrastructure deployment failed!'
        }
        cleanup {
            cleanWs()
        }
    }
}

// Define a helper method for AWS credentials
def withAWS(Closure body) {
    withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: 'AWS_SECRET_ACCESS_KEY'
    ]]) {
        sh """
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
            export AWS_REGION=$AWS_REGION
        """
        body()
    }
}
