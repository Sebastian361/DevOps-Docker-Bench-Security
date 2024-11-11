pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'docker-bench-security'
    }
    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/Sebastian361/DevOps-Docker-Bench-Security.git', branch: 'main'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                }
            }
        }
        stage('Run Security Checks') {
            steps {
                script {
                    sh 'docker run --rm --net host --pid host --cap-add audit_control -v /etc:/etc:ro -v /usr/bin/docker:/usr/bin/docker:ro -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro ${DOCKER_IMAGE}'
                }
            }
        }
        // Esta es la etapa que exporta las métricas a Prometheus
        stage('Export Metrics') {
            steps {
                script {
                    // Enviar métricas a Prometheus usando curl
                    sh 'curl -X POST http://localhost:9091/metrics/job/jenkins_pipeline'
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '**/resultados-seguridad-docker.txt', allowEmptyArchive: true
            echo 'Pipeline completo, revisa los resultados'
        }
    }
}
