pipeline {
    agent any
    tools {
        maven 'maven3'
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NEXUS_USER = 'admin'
        NEXUS_PASSWORD = 'admin123'
        //SNAP_REPO = 'maven-snapshot'
        RELEASE_REPO = 'myrepo-release'
        NEXUS_IP = '3.149.235.90'
        NEXUS_PORT = '8081'
        NEXUS_LOGIN = "nexus-cred" 
        
         }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main' , url: 'https://github.com/Jay-5454/springbootApp.git'
            }
        }
         stage('Versioning') {
    steps {
        script {
            sh 'mvn versions:set -DnewVersion=1.0.${BUILD_NUMBER}'
        }
    }
}
        stage('Maven Compile') {
            steps {
               echo 'Maven Compile Started'
               sh 'mvn compile'
            }
        } 
         stage('Maven Test') {
            steps {
               echo 'Maven Test Started'
               sh 'mvn test'
            }
        } 
        stage('File System Scan by Trivy') {
            steps {
               echo 'Trivy Scan Started'
               sh 'trivy fs --format table --output trivy-fs-output.txt .'
            }
        } 
        stage('Sonar Analysis') {
            steps {
               withSonarQubeEnv('sonar') {
                sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=SpringBoot -Dsonar.projectKey=SpringBoot \
                                                       -Dsonar.java.binaries=. -Dsonar.exclusions=**/trivy-fs-output.txt '''
               }
            }
        } 
        stage('Quality Gate') {
            steps {
              timeout(time: 1, unit: 'MINUTES') {
               waitForQualityGate abortPipeline: true, credentialsId: 'sonar'  
          }
        } 
      }
       stage('Maven Package') {
            steps {
               echo 'Maven package Started'
               sh 'mvn package'
          }
        } 
       stage("Jar Publish") {
            steps {
                nexusArtifactUploader(
                nexusVersion: 'nexus3',
                protocol: 'http',
                nexusUrl: "${NEXUS_IP}:${NEXUS_PORT}",
                groupId: 'TEST',
                version: '1.0',
                repository: "${RELEASE_REPO}",
                credentialsId: "${NEXUS_LOGIN}",
                artifacts: [
                    [artifactId: 'springbootApp',
                    classifier: '',
                    file: 'target/springbootApp.jar',
                    type: 'jar']
                ]
             )
    }
}
        stage('Build Docker Image and TAG') {
            steps {
                script {
                    // Build the Docker image using the renamed JAR file
                    script {
                            sh 'docker build -t springbootapp:latest .'
                        }
                }   
            }
        }
        stage('Docker Image Scan') {
            steps {
                sh 'trivy image --format table --scanners vuln -o trivy-image-report.html springbootapp:latest'
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                        sh "docker image tag springbootapp:latest jaycg54/springbootapp::latest"
                        sh "docker push jaycg54/springbootapp:latest"

                    }
                }
            }

            stage ('Push Docker Image to AWS ECR') {
        steps {
            script {
                sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 992382709064.dkr.ecr.us-east-2.amazonaws.com'
                sh 'docker tag myrepo:latest 992382709064.dkr.ecr.us-east-2.amazonaws.com/myrepo:latest'
                sh 'docker push 992382709064.dkr.ecr.us-east-2.amazonaws.com/myrepo:latest'
            }
        }
    }
    stage('Deploy To Kubernetes') {
        steps {
        withKubeConfig(caCertificate: '', clusterName: 'eks2', contextName: '', credentialsId: 'k8-cred', namespace: 'default', restrictKubeConfigAccess: false, serverUrl: 'https://A7C7808EC02839C9FBA67DFE109DBC33.gr7.us-east-1.eks.amazonaws.com')
         {
            sh "kubectl apply -f eks-deploy-k8s.yaml"
            }
        }
    }
    }
  }
