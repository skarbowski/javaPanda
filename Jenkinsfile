pipeline {
    agent {
        label 'Slave'
    }
    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
        terraform 'Terraform'
    }
    environment {
        IMAGE = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
        ANSIBLE = tool name: 'Ansible', type: 'com.cloudbees.jenkins.plugins.customtools.CustomTool'
    }
  
    stages {
        stage('Clear running apps') {
           steps {
               // Clear previous instances of app built
               sh 'docker rm -f pandaapp || true'
           }
        }
        stage('Get Code') {
            steps {
                // Get some code from a GitHub repository
                checkout scm
            }
        }
        stage('Build and Junit') {
            steps {
                // Run Maven on a Unix agent.
                sh "mvn clean install"
            }
        }
        stage('Build Docker image'){
            steps {
                sh "mvn package -Pdocker"
            }
        }
        stage('Run Docker app') {
            steps {
                sh "docker run -d -p 0.0.0.0:8080:8080 --name pandaapp -t ${IMAGE}:${VERSION}"
            }
        }
        //stage('Test Selenium') {
        //    steps {
        //        sh "mvn test -Pselenium"
        //    }
        //}
        stage('Deploy jar to artifactory') {
            steps {
                configFileProvider([configFile(fileId: '9d1ed313-ea70-4fa9-9934-7108c53eca75', variable: 'MAVEN_GLOBAL_SETTINGS')]) {
                    sh "mvn -gs $MAVEN_GLOBAL_SETTINGS deploy -Dmaven.test.skip=true -e"
                }
            } 
        }
        stage('Run terraform') {
            steps {
                dir('infrastructure/terraform') {      
                    withCredentials([file(credentialsId: 'aws_key', variable: 'terraformpanda')]) { sh "cp \$terraformpanda ../panda.pem" } 
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'panda']]) {
                    sh 'terraform init && terraform apply -var-file ./panda.tfvars -auto-approve '
                    } 
                }
            }
        }
        stage('Copy Ansible role') {
               steps {
                   sh 'sleep 180'
                   sh 'cp -r infrastructure/ansible/panda/ /etc/ansible/roles/'
                }
        }
        stage('Run Ansible') {
               steps {
                dir('infrastructure/ansible') {                
                    sh 'chmod 600 ../panda.pem'
                    sh 'ansible-playbook -i ./inventory playbook.yml -e ansible_python_interpreter=/usr/bin/python3'
                } 
            }
        }
        stage('Remove environment') { 
            steps {
                 input 'Remove environment'
                 dir('infrastructure/terraform') { 
                 withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS']]) {
                 sh 'terraform destroy -auto-approve -var-file panda.tfvars' 
                 } 
             } 
        } 
    }
    post {
        success {
            sh 'docker stop pandaapp'
            deleteDir()
        }
        failure {
            dir('infrastructure/terraform') { 
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws']]) {
                    sh 'terraform destroy -auto-approve -var-file panda.tfvars'
                }
            }
            sh 'docker stop pandaapp'
            deleteDir()
            }
        }
    }
}