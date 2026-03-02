pipeline {
    agent any

    parameters {
        booleanParam(name: 'ROLLBACK_ONLY', defaultValue: false, description: 'Откатиться на последнюю успешную версию?')
    }

    environment {
        IMAGE_NAME = "report-service"
        CONTAINER_NAME = "report-service"
        PORT = "8081"
        VERSION_FILE = "last_successful_build.txt"
    }

    stages {
        stage('Manual Rollback') {
            when { expression { params.ROLLBACK_ONLY } }
            steps {
                script {
                    performRollback()
                }
            }
        }

        stage('Test') {
            when { expression { !params.ROLLBACK_ONLY } }
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pytest test_app.py
                '''
            }
        }

        stage('Build') {
            when { expression { !params.ROLLBACK_ONLY } }
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Deploy') {
            when { expression { !params.ROLLBACK_ONLY } }
            steps {
                script {
                    deployImage("${IMAGE_NAME}:${BUILD_NUMBER}")
                }
            }
        }

        stage('Health-check') {
            when { expression { !params.ROLLBACK_ONLY } }
            steps {
                script {
                    try {
                        sleep 5
                        sh "curl -f http://localhost:${PORT}/health"
                        // Если curl успешен, сохраняем номер сборки как стабильный
                        sh "echo ${BUILD_NUMBER} > ${VERSION_FILE}"
                        echo "Health-check пройден. Версия ${BUILD_NUMBER} сохранена."
                    } catch (Exception e) {
                        echo "Health-check провален! Начинаю автоматический откат..."
                        performRollback()
                        error("Pipeline FAILED: Health-check не прошел.")
                    }
                }
            }
        }
    }
}

def deployImage(tag) {
    sh """
        docker stop ${CONTAINER_NAME} || true
        docker rm ${CONTAINER_NAME} || true
        docker run -d -p ${PORT}:5000 --name ${CONTAINER_NAME} ${IMAGE_NAME}:${tag}
    """
}

def performRollback() {
    if (fileExists(env.VERSION_FILE)) {
        def lastStable = readFile(env.VERSION_FILE).trim()
        echo "Откат к версии: ${lastStable}"
        deployImage(lastStable)
    } else {
        error "Файл ${env.VERSION_FILE} не найден. Откат невозможен!"
    }
}