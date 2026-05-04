pipeline {
    agent any

    parameters {
        choice(name: 'QualityGate', choices: ['Pass', 'Fail'], description: 'Fail pipeline on Quality Gate')
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ACCOUNT_ID = '475790160792'

        IMAGE_NAME = 'api'
        IMAGE_TAG = "${GIT_COMMIT.take(7)}-${BUILD_NUMBER}"

        ECR_DEV = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/api-dev"
        ECR_PROD = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/api-prod"

        HELM_REPO = 'https://github.com/snehaCodes276/application-helm-charts.git'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Static Analysis') {
            parallel {

                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('sonarqube') {
                            sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=api \
                            -Dsonar.sources=.
                            '''
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        sh '''
                        dependency-check.sh \
                        --project api \
                        --scan . \
                        --format XML \
                        --out .
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (params.QualityGate == 'Fail' && qg.status != 'OK') {
                        error "Quality Gate Failed"
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                sh 'pytest || true'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Grype Scan') {
            steps {
                sh "grype ${IMAGE_NAME}:${IMAGE_TAG} -o json > anchore-scan.json"

                script {
                    def critical = sh(script: "grep -c CRITICAL anchore-scan.json || true", returnStdout: true).trim()
                    if (critical.toInteger() > 0) {
                        error "Critical vulnerabilities found"
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    def repo = (env.BRANCH_NAME == 'master') ? ECR_PROD : ECR_DEV

                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${repo}:${IMAGE_TAG}
                    docker push ${repo}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Kubelinter Helm Check') {
            steps {
                sh """
                rm -rf application-helm-charts || true
                git clone ${HELM_REPO}

                kube-linter lint application-helm-charts/application-helmcharts/api
                """
            }
        }
    }

    post {
        always {
            sh 'kubectl get pods -A || true'
        }
    }
}
