def docker_image_name = 'my-shuttle'
def mysql_database_name = 'myshuttledb'

def docker_image = ''

pipeline {
    agent any

    parameters {
        string(name: 'ACR_HOST', defaultValue: 'holdemodev.azurecr.io', description: '')
        credentials(name: 'ACR_CREDENTIALS_ID', defaultValue: 'ACR', credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl', description: '')
        credentials(name: 'MYSQL_CREDENTIALS_ID', defaultValue: 'MuSQL', credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl', description: '')
        credentials(name: 'AZURE_SERVICE_PRINCIPAL_ID', defaultValue: 'ASP', credentialType: 'com.microsoft.azure.util.AzureCredentials', description: '')
        string(name: 'AZURE_RESOURCE_GROUP_NAME', defaultValue: 'HOL-Demo-Dev-Jenkins', description: '')
        string(name: 'AZURE_RESOURCE_NAME', defaultValue: 'hol-demo-dev-jenkins', description: '')
        string(name: 'AZURE_RESOURCE_LOCATION', defaultValue: 'West US', description: '')  
    }

    stages {
        
        stage('Maven package') {
            steps {
                sh 'mvn package -D Dtest=FaresTest,SimpleTest'
            }
        }
        
        stage('Docker build'){
            steps {
                script {
                    docker_image = docker.build(docker_image_name, '.')
                }
            }
        }
        
        stage('Docker push to ACR') {
            steps {
                script {
                    docker.withRegistry("https://${env.ACR_HOST}", params.ACR_CREDENTIALS_ID ) {
                        docker_image.tag("${env.BUILD_NUMBER}")
                        docker_image.push("${env.BUILD_NUMBER}");
                    }
                }
            }
        }
        
        stage('Terraform init and apply') {
            steps {            
                dir("terraform"){
                    sh "terraform init -no-color"
                    withCredentials([
                        usernamePassword(credentialsId: params.ACR_CREDENTIALS_ID, usernameVariable: 'ACR_USERNAME', passwordVariable: 'ACR_PASSWORD'),
                        usernamePassword(credentialsId: params.MYSQL_CREDENTIALS_ID, usernameVariable: 'MYSQL_USERNAME', passwordVariable: 'MYSQL_PASSWORD'),
                        azureServicePrincipal(credentialsId: params.AZURE_SERVICE_PRINCIPAL_ID, subscriptionIdVariable: 'ARM_SUBSCRIPTION_ID', clientIdVariable: 'ARM_CLIENT_ID', clientSecretVariable: 'ARM_CLIENT_SECRET', tenantIdVariable: 'ARM_TENANT_ID')
                    ]) {
                        sh ("terraform apply -auto-approve -no-color" 
                          + " -var 'azure={ resource_group_name=\"${params.AZURE_RESOURCE_GROUP_NAME}\", resource_name=\"${params.AZURE_RESOURCE_NAME}\", location=\"${params.AZURE_RESOURCE_LOCATION}\" }'" 
                          + " -var 'acr={ host=\"${params.ACR_HOST}\", username=\"$ACR_USERNAME\", password=\"$ACR_PASSWORD\", repository=\"$docker_image_name\", tag=${env.BUILD_NUMBER} }'" 
                          + " -var 'mysql={ username=\"$MYSQL_USERNAME\", password=\"$MYSQL_PASSWORD\", database=\"$mysql_database_name\" }'")
                    }
                }
            }
        }
        
        stage('Init MySQL database') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: params.MYSQL_CREDENTIALS_ID, usernameVariable: 'MYSQL_USERNAME', passwordVariable: 'MYSQL_PASSWORD')]) {
                        is_initialized = sh(returnStdout: true, script: "mysql -h ${params.AZURE_RESOURCE_NAME}.mysql.database.azure.com -u $MYSQL_USERNAME@${params.AZURE_RESOURCE_NAME} -p$MYSQL_PASSWORD &lt; ./src/db/isInitialized.sql") 
                        if (is_initialized.contains('0')) {
                            sh "mysql -h ${params.AZURE_RESOURCE_NAME}.mysql.database.azure.com -u $MYSQL_USERNAME@${params.AZURE_RESOURCE_NAME} -p$MYSQL_PASSWORD $mysql_database_name &lt; ./src/db/initData.sql"
                        }
                        else {
                            echo 'The database is already initialized.'
                        }
                    }
                }
            }
        }
        
    }
}