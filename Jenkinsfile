pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                echo "Клонирование репозитория Railway ERP System..."
                checkout scm
            }
        }
        
        stage('Build Info') {
            steps {
                echo "Сборка ветки: ${env.BRANCH_NAME}"
                sh '''
                    echo "Дата: $(date)"
                    echo "Коммит: ${GIT_COMMIT}"
                    echo "Автор: ${GIT_AUTHOR_NAME}"
                '''
            }
        }
        
        stage('Hello World') {
            steps {
                echo "🚂 Railway ERP System CI/CD работает!"
            }
        }
    }
    
    post {
        success {
            echo "✅ Сборка успешна!"
        }
        failure {
            echo "❌ Сборка неудачна!"
        }
    }
}
