pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'Maven3'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        REPOSITORY = '192.168.222.60:8083'
        IMAGE_REPO = 'fullstackapp'
    }
    stages {
        stage ('clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage ('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Louis-2b/DevOps-CICD-Full-stack-App.git'
            }
        }
        stage ('maven Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage ('maven Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage ('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' 
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Full-Stack-App \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=Full-Stack-App 
                    '''    
                }
            }
        }
        stage('quality gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SONAR-TOKEN'
                }
            }
        }
        stage ('Build war file') {
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }
        stage ('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage ('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER} ."
                }
            }
        }
        stage ('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'NEXUS-TOKEN', passwordVariable: 'PSW', usernameVariable: 'USER')]) {
                        sh "echo ${PSW} | docker login -u ${USER} --password-stdin ${REPOSITORY}"
                        sh "docker push ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER}"
                    }
                }
            }
        }
        stage('TRIVY') {
            steps {
                sh "trivy image ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER} > trivyimage.txt"
            }
        }
        stage ('Deploy to container') {
            steps {
                sh 'docker run -d --name bloggingapp -p 8080:8080 ${REPOSITORY}/${IMAGE_REPO}:${BUILD_NUMBER}'
            }
        }
        stage ('Deploy to kubernetes') {
            steps {
                dir ('kubernetes/development') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
                dir ('kubernetes/staging') {
                    script {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'K3S', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                            sh 'kubectl apply -f deployment.yaml'
                            sh 'kubectl apply -f ingress.yaml'
                            sh 'kubectl apply -f service.yaml'
                        }
                    }
                }
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'stephanetchanga2b@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}