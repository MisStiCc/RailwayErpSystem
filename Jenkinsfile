pipeline {
    agent any
    
    triggers {
        // Проверять изменения каждые 5 минут
        pollSCM('H/5 * * * *')

        // ИЛИ для GitHub вебхуков (раскомментировать если нужно)
        // githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Сборка запущена для ветки: ${env.BRANCH_NAME}"
                echo "📝 Коммит: ${GIT_COMMIT}"
                sh 'git log --oneline -3'
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Сборка Railway ERP System...'
                sh '''
                    echo "Текущая директория: $(pwd)"
                    echo "Дата: $(date)"
                    echo "Список файлов:"
                    ls -la
                '''

                script {
                    if (fileExists('compose.yaml')) {
                        echo '📁 Найден compose.yaml с PostgreSQL'
                        sh 'echo "--- Первые 10 строк compose.yaml ---" && head -10 compose.yaml'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Запуск тестов...'
                sh 'echo "Здесь будут запускаться тесты проекта"'

                // Пример команд для разных языков:
                // Для Java: sh 'mvn test'
                // Для Node.js: sh 'npm test'
                // Для Python: sh 'pytest'
                // Для .NET: sh 'dotnet test'
            }
        }

        stage('Deploy to Test') {
            when {
                branch 'main'  // Только для main ветки
            }
            steps {
                echo '🚚 Деплой на тестовый сервер...'
                sh 'echo "Здесь будет деплой на тестовое окружение"'
                // Пример: sh './deploy-test.sh'
            }
        }
    }

    post {
        always {
            echo '📊 ======== Сборка завершена ========'
            sh 'echo "Время выполнения: $BUILD_DURATION ms"'
        }
        success {
            echo '✅ ✅ ✅ Сборка успешна! ✅ ✅ ✅'
            // Можно добавить уведомления:
            // emailext to: 'team@example.com', subject: 'Сборка успешна', body: 'Поздравляем!'
            // slackSend channel: '#builds', message: "✅ Сборка ${env.JOB_NAME} успешна!"
        }
        failure {
            echo '❌ ❌ ❌ Сборка неуспешна! ❌ ❌ ❌'
            // Уведомление об ошибке
            // emailext to: 'team@example.com', subject: 'Сборка неуспешна', body: 'Требуется внимание!'
        }
        changed {
            echo '🔄 Статус сборки изменился!'
        }
        cleanup {
            echo '🧹 Очистка рабочего пространства...'
            // cleanWs()  // Раскомментировать для очистки workspace
        }
    }

    // Окружение - РАСКОММЕНТИРОВАТЬ ТОЛЬКО ЕСЛИ НУЖНЫ ПЕРЕМЕННЫЕ
    /*
    environment {
        PROJECT_NAME = 'RailwayERP'
        DEPLOY_ENV = 'test'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_URL = "${env.BUILD_URL}"
    }
    */
}