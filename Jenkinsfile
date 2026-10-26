pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node24'
    }
    options {
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        IMAGE_NAME    = 'harshjain5903/netflix'
        IMAGE_TAG     = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'netflix'
        NOTIFY_EMAIL  = 'harshjain5903@gmail.com'
    }
    stages {
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix'
                }
            }
        }
        stage('Quality gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Type check') {
            steps {
                sh 'npm run typecheck'
            }
        }
        stage('OWASP dependency check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS scan') {
            steps {
                sh 'trivy fs --scanners vuln,secret --format table -o trivyfs.txt .'
            }
        }
        stage('Docker build') {
            steps {
                withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                    sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_KEY -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
                }
            }
        }
        stage('Trivy image scan') {
            steps {
                // full report for the email, then fail the build on critical findings before anything is pushed
                sh 'trivy image --format table -o trivyimage.txt $IMAGE_NAME:$IMAGE_TAG'
                sh 'trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE_NAME:$IMAGE_TAG'
            }
        }
        stage('Docker push') {
            steps {
                withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                    sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                    sh 'docker push $IMAGE_NAME:latest'
                }
            }
        }
        stage('Deploy to container') {
            steps {
                sh 'docker rm -f netflix || true'
                sh 'docker run -d --name netflix -p 8081:8080 $IMAGE_NAME:$IMAGE_TAG'
            }
        }
        stage('Deploy to kubernetes') {
            steps {
                dir('Kubernetes') {
                    withKubeConfig(credentialsId: 'k8s') {
                        sh 'kubectl apply -f namespace.yml'
                        sh 'kubectl apply -f service.yml -f hpa.yml -f pdb.yml'
                        // render the build tag into the manifest so each release is its own rollout revision
                        sh 'sed "s#image: $IMAGE_NAME:latest#image: $IMAGE_NAME:$IMAGE_TAG#" deployment.yml | kubectl apply -f -'
                        sh '''
                            if ! kubectl rollout status deployment/netflix-app -n $K8S_NAMESPACE --timeout=180s; then
                                echo "Rollout failed, rolling back"
                                kubectl rollout undo deployment/netflix-app -n $K8S_NAMESPACE
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt,trivyimage.txt', allowEmptyArchive: true
            emailext attachLog: true,
                subject: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.currentResult}",
                body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                to: "${env.NOTIFY_EMAIL}",
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
