pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "juanpab/train-schedule"
    }
    stages {
        stage('SonarQube Analysis') {
            steps {
                script {
                  // debe estar configurado en Global Tool Configuration
                  scannerHome = tool 'sonarscanner'
                }
                withSonarQubeEnv('sonarqube') {
                  sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=application-test"
                }
            }
        }
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') { // timeout de espera al analisis
                    script {
                        def qg = waitForQualityGate()
                    }
                    if (qg.status != 'OK') {
                        error "Flujo detenido, no cumple los criterios de calidad y seguridad: ${qg.status}"
                    }
                }
            }
        }
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('ScanDockerImage') {
            when {
                branch 'master'
            }
            steps {
                aquaMicroscanner imageName: DOCKER_IMAGE_NAME, notCompliesCmd: '', onDisallowed: 'ignore', outputFormat: 'html'
            }
        }
        stage('PushDockerImage') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = 0
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}
