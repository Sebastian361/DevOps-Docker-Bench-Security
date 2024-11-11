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
                }
            }
        }
        stage('Verify Docker Access') {
            steps {
                script {
                    // Verificar que Jenkins tiene acceso a Docker
                    sh 'docker info'
                }
            }
        }
        stage('Deploy Nginx') {
            steps {
                script {
                    try {
                        // Leer el puntaje del Docker Bench Security
                        def score = sh(script: "grep -oP '(?<=Score: )\d+' resultados-seguridad-docker.txt", returnStdout: true).trim()

                        // Imprimir el puntaje de seguridad
                        echo "Docker Bench Security Score: ${score}"

                        // Validar si el puntaje es adecuado (mayor o igual a 3)
                        if (score.toInteger() >= 3) {
                            echo "El puntaje es adecuado. Procediendo con el despliegue del contenedor."

                            // Desplegar la imagen de Nginx con permisos elevados
                            def containerId = sh(script: 'docker run -d --privileged --name nginx-container -p 80:80 nginx', returnStdout: true).trim()
                            echo "Contenedor Nginx desplegado con ID: ${containerId}"

                            // Verificar los contenedores en ejecución
                            def containersRunning = sh(script: 'docker ps', returnStdout: true).trim()
                            echo "Contenedores en ejecución: ${containersRunning}"

                            // Capturar logs del contenedor Nginx
                            def logs = sh(script: 'docker logs nginx-container', returnStdout: true).trim()
                            echo "Logs del contenedor Nginx: ${logs}"
                        } else {
                            echo "El puntaje de seguridad es bajo. No se realizará el despliegue."
                        }
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
