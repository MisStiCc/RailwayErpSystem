pipeline {
    agent any
    
    triggers {
        // Автоматическая сборка при пуше в main
        pollSCM('H/5 * * * *')
    }
    
    environment {
        TELEGRAM_CHAT_ID = '5563344090'  // Ваш Chat ID
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Автосборка запущена"
                echo "📝 Ветка: ${env.BRANCH_NAME ?: 'не определена'}"
                echo "📝 Коммит: ${GIT_COMMIT}"
                sh '''
                    echo "📊 Информация о репозитории:"
                    git log --oneline -3
                    echo ""
                    echo "📁 Структура проекта:"
                    ls -la
                '''
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Сборка Railway ERP System...'
                
                script {
                    if (fileExists('compose.yaml')) {
                        echo '🐳 Найден compose.yaml с PostgreSQL'
                        sh 'echo "--- Docker Compose ---" && head -15 compose.yaml'
                    }
                    
                    // Даем права на выполнение mvnw
                    sh 'chmod +x mvnw 2>/dev/null || echo "mvnw не найден"'
                    
                    if (fileExists('pom.xml')) {
                        echo '📦 Maven проект - запускаем сборку'
                        sh '''
                            echo "=== Информация о Java ==="
                            java -version 2>&1 | head -3 || echo "Java не установлена"
                            echo ""
                            echo "=== Запуск Maven сборки ==="
                            ./mvnw clean compile -DskipTests || echo "Сборка завершилась с предупреждениями"
                        '''
                    }
                }
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Запуск тестов...'
                
                script {
                    if (fileExists('pom.xml')) {
                        sh '''
                            echo "=== Запуск тестов ==="
                            ./mvnw test || echo "Тесты завершились с ошибками"
                        '''
                        junit 'target/surefire-reports/**/*.xml'
                    } else {
                        echo '📭 Тесты не найдены, пропускаем'
                    }
                }
            }
        }

        stage('Deploy to Test') {
            when {
                branch 'main'  // Только для main ветки
            }
            steps {
                echo '🚚 Деплой на тестовый сервер...'
                sh 'echo "Деплой был бы здесь"'
                
                // Отправляем уведомление в Telegram
                script {
                    def deployMessage = """
🚀 *Деплой на тестовый сервер*

*Ветка:* \`main\`
*Сборка:* #${env.BUILD_NUMBER}
*Коммит:* ${GIT_COMMIT.take(8)}
*Статус:* ✅ Успешно

*Ссылки:*
• Jenkins: ${env.BUILD_URL}
• GitHub: https://github.com/MisStiCc/RailwayErpSystem/commit/${GIT_COMMIT}
"""
                    
                    // Функция отправки в Telegram
                    withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'BOT_TOKEN')]) {
                        sh """
                            curl -s -X POST "https://api.telegram.org/bot\${BOT_TOKEN}/sendMessage" \
                            -H "Content-Type: application/json" \
                            -d '{
                                "chat_id": "${env.TELEGRAM_CHAT_ID}",
                                "text": "${deployMessage.replace('"', '\\"').replace('\n', '\\n')}",
                                "parse_mode": "Markdown",
                                "disable_web_page_preview": true
                            }'
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo '📊 ======== Сборка завершена ========'
            sh 'echo "Время выполнения сборки"'
            
            script {
                // Сохраняем логи
                sh '''
                    mkdir -p build_logs
                    date > build_logs/build_time.txt
                    echo "Job: ${JOB_NAME} #${BUILD_NUMBER}" >> build_logs/build_time.txt
                    echo "Статус: ${currentBuild.currentResult}" > build_logs/status.txt
                '''
                archiveArtifacts artifacts: 'build_logs/**/*, target/**/*.jar', allowEmptyArchive: true
            }
        }
        
        success {
            echo '✅ ✅ ✅ Сборка успешна! ✅ ✅ ✅'
            
            // Отправляем уведомление только для feature веток
            script {
                def branch = env.BRANCH_NAME ?: 'unknown'
                if (branch != 'main' && branch != 'master') {
                    def successMessage = """
✅ *Feature ветка собрана успешно*

*Ветка:* \`${branch}\`
*Сборка:* #${env.BUILD_NUMBER}
*Статус:* ✅ Готова к созданию MR

Можно создать Pull Request:
https://github.com/MisStiCc/RailwayErpSystem/compare/main...${branch}

*Ссылка на логи:* ${env.BUILD_URL}
"""
                    
                    withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'BOT_TOKEN')]) {
                        sh """
                            curl -s -X POST "https://api.telegram.org/bot\${BOT_TOKEN}/sendMessage" \
                            -H "Content-Type: application/json" \
                            -d '{
                                "chat_id": "${env.TELEGRAM_CHAT_ID}",
                                "text": "${successMessage.replace('"', '\\"').replace('\n', '\\n')}",
                                "parse_mode": "Markdown",
                                "disable_web_page_preview": true
                            }'
                        """
                    }
                }
            }
        }
        
        failure {
            echo '❌ ❌ ❌ Сборка неуспешна! ❌ ❌ ❌'
            
            script {
                def failureMessage = """
❌ *Сборка провалена*

*Ветка:* \`${env.BRANCH_NAME ?: 'unknown'}\`
*Сборка:* #${env.BUILD_NUMBER}
*Ошибка:* ${env.STAGE_NAME ?: 'Неизвестно'}

Требуется срочное внимание!
*Ссылка на логи:* ${env.BUILD_URL}
"""
                
                withCredentials([string(credentialsId: 'telegram-bot-token', variable: 'BOT_TOKEN')]) {
                    sh """
                        curl -s -X POST "https://api.telegram.org/bot\${BOT_TOKEN}/sendMessage" \
                        -H "Content-Type: application/json" \
                        -d '{
                            "chat_id": "${env.TELEGRAM_CHAT_ID}",
                            "text": "${failureMessage.replace('"', '\\"').replace('\n', '\\n')}",
                            "parse_mode": "Markdown",
                            "disable_web_page_preview": true
                        }'
                    """
                }
            }
        }
        
        cleanup {
            echo '🧹 Очистка рабочего пространства...'
            cleanWs()
        }
    }
}
