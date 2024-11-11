pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'docker-bench-security'
        NGINX_IMAGE = 'nginx'
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
                    sh '''
                        docker run --rm --net host --pid host --userns host --cap-add audit_control \
                        -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
                        -v /etc:/etc:ro \
                        -v /lib/systemd/system:/lib/systemd/system:ro \
                        -v /usr/bin/containerd:/usr/bin/containerd:ro \
                        -v /usr/bin/runc:/usr/bin/runc:ro \
                        -v /usr/lib/systemd:/usr/lib/systemd:ro \
                        -v /var/lib:/var/lib:ro \
                        -v /var/run/docker.sock:/var/run/docker.sock:ro \
                        --label docker_bench_security \
                        ${DOCKER_IMAGE} > resultados-seguridad-docker.txt

                        echo "<html><head><title>Reporte de Seguridad Docker</title></head><body><pre>" > resultados-seguridad-docker.html
                        cat resultados-seguridad-docker.txt >> resultados-seguridad-docker.html
                        echo "</pre></body></html>" >> resultados-seguridad-docker.html
                    '''
                }
            }
        }
        stage('Deploy Nginx') {
            steps {
                script {
                    try {
                        echo "Iniciando despliegue de Nginx..."
                        def containerId = sh(script: 'docker run -d --privileged --name nginx-container -p 80:80 nginx', returnStdout: true).trim()
                        echo "Contenedor Nginx desplegado con ID: ${containerId}"
                        sh 'docker ps -a'
                    } catch (Exception e) {
                        echo "Error al desplegar el contenedor Nginx: ${e}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '**/resultados-seguridad-docker.html', allowEmptyArchive: true
            echo 'Pipeline completo, revisa los resultados en HTML'
        }
    }
}
