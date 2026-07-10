pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        AWS_REGION = 'ap-southeast-2'
        CLUSTER_NAME = 'springboot-app'
    }

    stages {

        stage('Checkout Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/balu-143/Devsecops-spring-boot'
            }
        }

        stage('Build & JUnit Test') {
            steps {
                sh 'mvn clean install'
            }
            post {
                success {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=jenkins-project_211221
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                    -t baluhub/spring-boot-app:v1.${BUILD_NUMBER} .

                    docker tag \
                    baluhub/spring-boot-app:v1.${BUILD_NUMBER} \
                    baluhub/spring-boot-app:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo $DOCKER_PASS | docker login \
                        -u $DOCKER_USER \
                        --password-stdin

                        docker push baluhub/spring-boot-app:v1.${BUILD_NUMBER}
                        docker push baluhub/spring-boot-app:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    mkdir -p /var/lib/jenkins/trivy-cache
                    mkdir -p /var/lib/jenkins/trivy-tmp

                    export TMPDIR=/var/lib/jenkins/trivy-tmp

                    trivy image \
                        --cache-dir /var/lib/jenkins/trivy-cache \
                        --format template \
                        --template "@/opt/trivy/templates/html.tpl" \
                        -o report.html \
                        baluhub/spring-boot-app:latest
                '''
            }
        }

        stage('Upload Scan Report to AWS S3') {
            steps {
                sh '''
                    aws s3 cp report.html \
                    s3://trivy-scan-image/${JOB_NAME}/${BUILD_NUMBER}/report.html
                '''
            }
        }

        stage('Configure AWS for EKS') {
            steps {
                
                    sh '''
                        export AWS_DEFAULT_REGION=$AWS_REGION

                        aws sts get-caller-identity

                        aws eks update-kubeconfig \
                            --region $AWS_REGION \
                            --name $CLUSTER_NAME
                    '''
                
            }
        }

        stage('Verify EKS Connection') {
            steps {
                sh '''
                    kubectl get nodes
                    kubectl get pods -A
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl apply -f spring-boot-deployment.yaml
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }

        always {
            archiveArtifacts artifacts: 'report.html',
                             allowEmptyArchive: true
        }
    }
}
