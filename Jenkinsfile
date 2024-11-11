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
                    // Leer el archivo de texto y eliminar caracteres ANSI
                    def result = readFile('resultados-seguridad-docker.txt')
                    
                    // Eliminar caracteres ANSI (colores y formateo) con una expresión regular
                    def cleanResult = result.replaceAll(/\x1b\[[0-9;]*m/, '')

                    // Buscar la línea que contiene "Score:"
                    def scoreLine = cleanResult.readLines().find { it.contains("Score:") }
                    def score = scoreLine?.split(":")?.last()?.trim()?.toInteger()

                    echo "Docker Bench Security Score: ${score}"

                    // Validar si el puntaje es mayor o igual a 3
                    if (score >= 3) {
                        echo "El puntaje es adecuado. Procediendo con el despliegue del contenedor."
                    } else {
                        error "El puntaje de seguridad es bajo (${score}). No se realizará el despliegue."
                    }
                }
            }
        }
        stage('Verify Docker Access') {
            steps {
                script {
                    try {
                        echo "Verificando acceso a Docker..."
                        // Comando para verificar si Docker está disponible en Jenkins
                        def dockerInfo = sh(script: 'docker info', returnStdout: true).trim()
                        echo "Docker Info: ${dockerInfo}"
                    } catch (e) {
                        echo "Error al acceder a Docker: ${e}"
                        currentBuild.result = 'FAILURE' // Marca el build como fallido
                    }
                }
            }
        }
        stage('Deploy Nginx') {
            when {
                expression { return score >= 3 } // Solo despliega si el puntaje es adecuado
            }
            steps {
                script {
                    try {
                        echo "Verificando y eliminando cualquier contenedor existente de Nginx..."
                        // Detener y eliminar el contenedor si ya existe
                        sh '''
                            if [ "$(docker ps -aq -f name=nginx-container)" ]; then
                                docker rm -f nginx-container
                            fi
                        '''
                        
                        echo "Desplegando la imagen de Docker Nginx..."
                        // Ejecutar el contenedor de Nginx con el flag --privileged
                        def containerId = sh(script: 'docker run -d --privileged --name nginx-container -p 80:80 nginx', returnStdout: true).trim()
                        echo "Contenedor Nginx desplegado con ID: ${containerId}"
                    } catch (e) {
                        echo "Error al desplegar el contenedor Nginx: ${e}"
                        currentBuild.result = 'FAILURE' // Marca el build como fallido
                    }
                }
            }
        }
        stage('Export Metrics') {
            steps {
                script {
                    try {
                        echo "Enviando métricas a Prometheus..."
                        // Enviar métricas a Prometheus usando curl
                        sh 'curl -X POST http://localhost:9091/metrics/job/jenkins_pipeline'
                    } catch (e) {
                        echo "Error al exportar métricas a Prometheus: ${e}"
                        currentBuild.result = 'FAILURE' // Marca el build como fallido
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
