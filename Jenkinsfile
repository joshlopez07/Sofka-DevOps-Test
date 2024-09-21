pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/joshlopez07/Sofka-DevOps-Test.git' // Repositorio de Node.js
        BRANCH = 'main'
        GITHUB_CREDENTIALS = 'github_credentials'
        OWASP_REPORT_PATH = 'owasp-report.html'
        SONAR_PROJECT_KEY = 'joshlopez07_Sofka-DevOps-Test'
        SONAR_ORG = 'Joseph López'
        DOCKER_IMAGE = "joshlopez07/Sofka-DevOps-Test:1.0.0" // Repositorio en Docker Hub
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        MINIKUBE_IP = '3.85.207.165' // IP instancia EC2
        KUBECONFIG = '/home/jenkins/.kube/config'
        NVD_API_KEY = 'c04ad272-f369-4fc3-9171-820a44bfb756'
        /*JMETER_HOME = '/opt/jmeter'  // Ruta donde está instalado JMeter
        JMETER_JMX = 'Sofka-DevOps-Test.jmx'                // Nombre del archivo .jmx
        RESULTS_DIR = 'jmeter_results'                     // Carpeta para almacenar resultados de JMeter*/
    }

    stages {
        stage('Clone Code') {
            steps {
                git branch: "${BRANCH}", url: "${GIT_REPO}", credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }

        stage('Install Dependencies') {
            steps {
                //sh 'nvm install 10.16.3'
                sh 'npm install'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('Test OWASP') {
            steps {
                sh 'npm audit --audit-level=high'  // Realiza un escaneo de seguridad con npm audit
                // Si tienes OWASP Dependency Check configurado para Node.js, puedes agregar lo siguiente
                sh 'dependency-check --project "Sofka-DevOps-Test" --out ./dependency-check-report.html --scan ./ --nvdApiKey ${NVD_API_KEY}'
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'OWASP Dependency-Check Vulnerabilities'
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Test Code Review') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    // Instalar SonarScanner y ejecutar escaneo con cobertura
                    sh 'npm install -g sonar-scanner'
                    sh 'npm install nyc --save-dev'
                    sh 'jest --coverage' // o `npm test` si estás usando Mocha + NYC

                    // Configurar SonarScanner para incluir el informe de cobertura
                    sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                    -Dsonar.organization=${SONAR_ORG} \
                    -Dsonar.sources=./src \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t Sofka-DevOps-Test:1.0.0 .'
                    sh "docker tag Sofka-DevOps-Test:1.0.0 ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Scan Docker Image with Trivy') {
            steps {
                script {
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --severity HIGH,CRITICAL --no-progress --exit-code 0 ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", passwordVariable: 'DOCKERHUB_PASSWORD', usernameVariable: 'DOCKERHUB_USERNAME')]) {
                        sh 'echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin'
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                script {
                    sh "kubectl apply -f deployment.yaml --kubeconfig=${KUBECONFIG}"
                }
            }
        }

        /*stage('JMeter Test') {
            steps {
                script {
                    sh 'rm -rf jmeter_results/*'
                    sh "mkdir -p ${RESULTS_DIR}"
                    sh "${JMETER_HOME}/bin/jmeter -n -t ${JMETER_JMX} -l ${RESULTS_DIR}/results.jtl -e -o ${RESULTS_DIR}/report"
                }
            }
        }*/
    }

    post {
        /*always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: "${RESULTS_DIR}/report",
                reportFiles: 'index.html',
                reportName: 'JMeter Test Report'
            ])
        }*/
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}