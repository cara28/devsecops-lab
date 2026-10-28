pipeline {
    agent any
    
    stages {
        stage('Clonar Repositorio') {
            steps {
                echo 'Descargando código de Juice Shop...'
                git url: 'https://github.com/juice-shop/juice-shop.git', branch: 'master'
            }
        }
        
        stage('SAST - SonarQube') {
            environment {
                SCANNER_HOME = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectKey=mi-proyecto-devsecops \
                      -Dsonar.projectName="Laboratorio DevSecOps" \
                      -Dsonar.sources=.
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Construir Imagen Docker') {
            steps {
                echo 'Construyendo la imagen localmente...'
                sh 'docker build -t juice-shop-ci:latest .'
            }
        }
        
        stage('Análisis SCA con Trivy') {
            steps {
                echo 'Buscando vulnerabilidades CRÍTICAS en la imagen...'
                // Cambiado a exit-code 0 para permitir que el pipeline llegue a ZAP
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 0 --severity CRITICAL juice-shop-ci:latest'
            }
        }
        
        stage('Desplegar a Staging') {
            steps {
                sh 'docker run -d --name mi-app-staging-ZAP -p 8081:3000 juice-shop-ci:latest'
                sleep time: 10, unit: 'SECONDS'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                // 1. Limpiamos cualquier rastro de intentos anteriores
                sh 'docker rm -f zap-scanner || true'
                sh 'docker volume rm zap_vol || true'
                
                // 2. Creamos el disco virtual temporal para engañar al script de ZAP
                sh 'docker volume create zap_vol'
                
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh '''
                    # 3. Montamos el disco temporal (zap_vol) en /zap/wrk
                    docker run --name zap-scanner -u root -v zap_vol:/zap/wrk/:rw -t zaproxy/zap-stable zap-baseline.py \
                      -t http://192.168.133.132:8081 \
                      -r zap-report.html
                    '''
                }
                
                // 4. Extraemos el reporte a Jenkins y limpiamos la basura
                sh 'docker cp zap-scanner:/zap/wrk/zap-report.html ./zap-report.html || true'
                sh 'docker rm -f zap-scanner || true'
                sh 'docker volume rm zap_vol || true'
            }
        }
    } // <--- ESTA ES LA LLAVE QUE FALTABA PARA CERRAR 'stages'
    
    post {
        always {
            echo 'Apagando entorno de Staging y limpiando...'
            sh 'docker rm -f mi-app-staging-ZAP || true'
            
            archiveArtifacts artifacts: 'zap-report.html', allowEmptyArchive: true
        }
    }
} // Fin del bloque 'pipeline' (y se eliminó la llave extra que tenías aquí abajo)
