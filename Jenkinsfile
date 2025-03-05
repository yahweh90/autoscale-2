pipeline {
    agent any
    
    environment {
        AWS_REGION = 'us-east-1'
        TERRAFORM_DIR = '.'
        GITHUB_REPO_URL = 'https://github.com/yahweh90/autoscale-2'
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
                    url: "${GITHUB_REPO_URL}"
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
// pipeline{
//     agent any
//     tools {
//         jfrog 'jfrog-cli'
//     }
//     stages {
//         stage ('Testing') {
//             steps {
//                 jf '-v' 
//                 jf 'c show'
//                 jf 'rt ping'
//                 sh 'touch test-file'
//                 jf 'rt u test-file jfrog-cli/'
//                 jf 'rt bp'
//                 jf 'rt dl jfrog-cli/test-file'
//             }
//         } 
//     }
// }
pipeline {
    agent any
    
    tools {
        terraform 'terraform-latest'
        jfrog 'jfrog-cli'
    }
    
    environment {
        AWS_REGION = 'us-east-1'
        TF_IN_AUTOMATION = 'true'
    }
    
    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
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
                    url: 'https://github.com/yahweh90/autoscale-2',
                    changelog: true
            }
        }

        stage('Initialize Terraform') {
            steps {
                sh 'terraform --version'
                sh 'terraform init -no-color'
            }
        }

        stage('Plan Terraform') {
            steps {
                withAWS(credentials: 'AWS_Credentials', region: env.AWS_REGION) {
                    sh '''
                        terraform plan -no-color -out=tfplan
                        terraform show -no-color tfplan > tfplan.txt
                    '''
                    archiveArtifacts artifacts: 'tfplan.txt', fingerprint: true
                }
            }
        }

        stage('Upload Plan to JFrog') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER
                    def timestamp = sh(script: 'date +%Y%m%d_%H%M%S', returnStdout: true).trim()
                    def planFileName = "tfplan_${buildNumber}_${timestamp}.txt"
                    
                    sh "cp tfplan.txt ${planFileName}"
                    jf "rt u ${planFileName} terraform-plans/"
                    
                    // Optional: Add properties to the uploaded file
                    jf """rt sp terraform-plans/${planFileName} \
                        "build.number=${buildNumber}" \
                        "build.timestamp=${timestamp}" \
                        "terraform.environment=${env.AWS_REGION}"
                    """
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Review the plan in tfplan.txt. Approve Terraform Apply?", ok: "Deploy"
                withAWS(credentials: 'AWS_Credentials', region: env.AWS_REGION) {
                    sh 'terraform apply -no-color -auto-approve tfplan'
                }
            }
        }
    }
    
    post {
        always {
            cleanWs(cleanWhenNotBuilt: false,
                   deleteDirs: true,
                   disableDeferredWipeout: true,
                   notFailBuild: true)
        }
        success {
            echo 'Terraform deployment completed successfully!'
        }
        failure {
            echo 'Terraform deployment failed!'
        }
    }
}
