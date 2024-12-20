pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'docker-bench-security'
        NGINX_IMAGE = 'nginx'  // Imagen de Nginx para el despliegue
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
        stage('Verify Docker Access') {
            steps {
                script {
                    try {
                        echo "Verificando acceso a Docker..."
                        def dockerInfo = sh(script: 'docker info', returnStdout: true).trim()
                        echo "Docker Info: ${dockerInfo}"
                    } catch (e) {
                        echo "Error al acceder a Docker: ${e}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Deploy Nginx') {
            steps {
                script {
                    try {
                        echo "Verificando si existe un contenedor anterior de Nginx..."
                        sh '''
                            if [ "$(docker ps -aq -f name=nginx-container)" ]; then
                                docker rm -f nginx-container
                            fi
                        '''
                        
                        echo "Desplegando un nuevo contenedor de Nginx..."
                        def containerId = sh(script: 'docker run -d --privileged --name nginx-container -p 80:80 nginx', returnStdout: true).trim()
                        echo "Contenedor Nginx desplegado con ID: ${containerId}"
                    } catch (e) {
                        echo "Error al desplegar el contenedor Nginx: ${e}"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Export Metrics') {
            steps {
                script {
                    try {
                        echo "Enviando métricas a Prometheus..."
                        sh 'curl -X POST http://localhost:9091/metrics/job/jenkins_pipeline'
                    } catch (e) {
                        echo "Error al exportar métricas a Prometheus: ${e}"
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
