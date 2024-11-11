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
                    // Ejecutar el script de docker-bench-security
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

                        # Generar el archivo HTML con los resultados
                        echo "<html><head><title>Reporte de Seguridad Docker</title></head><body><pre>" > resultados-seguridad-docker.html
                        cat resultados-seguridad-docker.txt >> resultados-seguridad-docker.html
                        echo "</pre></body></html>" >> resultados-seguridad-docker.html
                    '''
                    // Leer el puntaje desde el archivo de resultados
                    def result = readFile('resultados-seguridad-docker.txt')
                    def score = result.find(/Total score: (\d+)/) { match, number -> number.toInteger() }
                    echo "Docker Bench Security Score: ${score}"

                    // Validar si el puntaje es mayor o igual a 5
                    if (score >= 5) {
                        echo "El puntaje es adecuado. Procediendo con el despliegue del contenedor."
                    } else {
                        error "El puntaje de seguridad es bajo (${score}). No se realizará el despliegue."
                    }
                }
            }
        }
        stage('Deploy Nginx') {
            when {
                expression { return score >= 5 } // Solo despliega si el puntaje es adecuado
            }
            steps {
                script {
                    echo "Desplegando la imagen de Docker Nginx..."
                    // Ejecutar el contenedor de Nginx
                    sh 'docker run -d --name nginx-container -p 80:80 nginx'
                    echo "Nginx ha sido desplegado."
                }
            }
        }
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
            archiveArtifacts artifacts: '**/resultados-seguridad-docker.html', allowEmptyArchive: true
            echo 'Pipeline completo, revisa los resultados en HTML'
        }
    }
}
