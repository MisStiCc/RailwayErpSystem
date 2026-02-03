pipeline {
    agent any
    
    parameters {
        booleanParam(name: 'SKIP_TESTS', defaultValue: false)
        booleanParam(name: 'SEND_TELEGRAM', defaultValue: true)
        choice(name: 'ENVIRONMENT', choices: ['dev', 'test', 'prod'])
    }

    stages {
        stage('🚀 Начало') {
            steps {
                script {
                    echo "=== Railway ERP System (Spring Boot) ==="
                    echo "Сборка #${BUILD_NUMBER}"
                    echo "Ветка: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('📥 Код') {
            steps {
                checkout scm
            }
        }

        stage('🔧 Проверка') {
            steps {
                script {
                    sh '''
                        echo "Инструменты:"
                        java -version
                        mvn -version || echo "Maven не установлен"
                        echo ""
                        echo "Файлы проекта:"
                        ls -la
                    '''
                }
            }
        }

        stage('🏗️ Сборка Maven') {
            steps {
                script {
                    def skipTests = params.SKIP_TESTS ? '-DskipTests' : ''
                    sh """
                        echo "Сборка Spring Boot приложения..."
                        ./mvnw clean compile ${skipTests} || mvn clean compile ${skipTests}
                    """
                }
            }
        }

        stage('🧪 Тестирование') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                sh './mvnw test || mvn test'
            }
        }

        stage('📦 Пакет') {
            steps {
                sh './mvnw package -DskipTests || mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            echo "✅ Сборка #${BUILD_NUMBER} успешна!"
        }
        failure {
            echo "❌ Сборка #${BUILD_NUMBER} упала!"
        }
    }
}