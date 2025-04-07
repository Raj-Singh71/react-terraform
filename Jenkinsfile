pipeline {
    agent any

    environment {
        AZURE_CREDENTIALS_ID = 'jenkins-sp'
        RESOURCE_GROUP = 'rg-react'
        APP_SERVICE_NAME = 'reactwebappjenkins838796'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/PawanK7390/react-azure-deploy.git'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('terraform') {
                    bat 'terraform init || exit /b'
                }
            }
        }

        stage('Terraform Import') {
            steps {
                dir('terraform') {
                    bat '''
                        terraform import azurerm_resource_group.rg "/subscriptions/eea7dd66-806c-47a7-912f-2e3f1af71f5e/resourceGroups/rg-react" || exit /b
                        terraform import azurerm_service_plan.asp "/subscriptions/eea7dd66-806c-47a7-912f-2e3f1af71f5e/resourceGroups/rg-react/providers/Microsoft.Web/serverFarms/react-app-plan" || exit /b
                        terraform import azurerm_linux_web_app.react_app "/subscriptions/eea7dd66-806c-47a7-912f-2e3f1af71f5e/resourceGroups/rg-react/providers/Microsoft.Web/sites/reactwebappjenkins838796" || exit /b
                    '''
                }
            }
        }

        stage('Terraform Plan & Apply') {
            steps {
                dir('terraform') {
                    bat 'terraform plan -out=tfplan || exit /b'
                    bat 'terraform apply -auto-approve tfplan || exit /b'
                }
            }
        }

        stage('Install Dependencies & Build React') {
            steps {
                bat 'npm install || exit /b'
                bat 'npm run build || exit /b'
            }
        }

        stage('Check Build Folder') {
            steps {
                script {
                    def buildExists = fileExists('build\\index.html')
                    if (!buildExists) {
                        error("Build folder missing or empty. Make sure 'npm run build' succeeded.")
                    }
                }
            }
        }

        stage('Deploy to Azure using az webapp deploy') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: "${AZURE_CREDENTIALS_ID}")]) {
                    bat 'echo Logging into Azure...'
                    bat 'az login --service-principal -u %AZURE_CLIENT_ID% -p %AZURE_CLIENT_SECRET% --tenant %AZURE_TENANT_ID%'
                    bat 'az account set --subscription %AZURE_SUBSCRIPTION_ID%'

                    bat 'az webapp config appsettings set --resource-group %RESOURCE_GROUP% --name %APP_SERVICE_NAME% --settings SCM_DO_BUILD_DURING_DEPLOYMENT=false'
                    bat 'az webapp config appsettings set --resource-group %RESOURCE_GROUP% --name %APP_SERVICE_NAME% --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true'

                    bat 'echo Deploying React app using build/ folder...'
                    bat 'az webapp deploy --resource-group %RESOURCE_GROUP% --name %APP_SERVICE_NAME% --src-path build --type static || exit /b'

                    bat 'echo Restarting App Service...'
                    bat 'az webapp restart --resource-group %RESOURCE_GROUP% --name %APP_SERVICE_NAME%'
                }
            }
        }
    }

    post {
        success {
            echo ' React App Deployed Successfully using az webapp deploy!'
        }
        failure {
            echo ' Deployment Failed. Check logs and troubleshoot.'
        }
        always {
            cleanWs()
        }
    }
}
