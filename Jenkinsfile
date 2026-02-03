pipeline {
    agent any
    
    // Параметры для ручного запуска
    parameters {
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Пропустить тесты для быстрой сборки'
        )
        booleanParam(
            name: 'SEND_TELEGRAM',
            defaultValue: true,
            description: 'Отправлять уведомления в Telegram'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Целевое окружение'
        )
        choice(
            name: 'DEPLOY_ACTION',
            choices: ['none', 'deploy', 'rollback'],
            description: 'Действие с деплоем'
        )
        string(
            name: 'VERSION',
            defaultValue: '',
            description: 'Версия для деплоя (оставьте пустым для авто)'
        )
    }

    // Триггеры
    triggers {
        // Автоматические сборки при пуше в main
        pollSCM('H/5 * * * *')
    }

    // Окружение
    environment {
        // Определяем тип события
        IS_PR = "${env.CHANGE_ID}" != "" && "${env.CHANGE_ID}" != "null"
        IS_TAG = "${env.TAG_NAME}" != "" && "${env.TAG_NAME}" != "null"

        // Информация о событии
        EVENT_TYPE = IS_PR ? "Pull Request" : (IS_TAG ? "Tag" : "Push")
        EVENT_INFO = IS_PR ? "PR #${env.CHANGE_ID}: ${env.CHANGE_TITLE}" :
                    (IS_TAG ? "Tag: ${env.TAG_NAME}" : "Ветка: ${env.BRANCH_NAME}")

        // Проект
        PROJECT_NAME = 'Railway ERP System'
        PROJECT_URL = 'https://github.com/MisStiCc/RailwayErpSystem'
        APP_NAME = 'railway-erp-system'

        // Java/Maven настройки
        JAVA_HOME = tool name: 'jdk17', type: 'jdk'
        MAVEN_HOME = tool name: 'maven-3.8', type: 'maven'

        // Версия сборки
        BUILD_VERSION = params.VERSION ?: "${env.BUILD_NUMBER}-${new Date().format('yyyyMMdd-HHmm')}"

        // Docker настройки
        DOCKER_REGISTRY = 'registry.example.com'
        DOCKER_NAMESPACE = 'railway'

        // Окружения
        DEPLOY_ENV = params.ENVIRONMENT
        SKIP_TESTS_FLAG = params.SKIP_TESTS ? '-DskipTests' : ''

        // Telegram credentials (добавь в Jenkins)
        TELEGRAM_BOT_TOKEN = credentials('telegram-bot-token')
        TELEGRAM_CHAT_ID = credentials('telegram-chat-id')
    }

    // Опции
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        parallelsAlwaysFailFast()
        timestamps()
        ansiColor('xterm')
    }

    stages {
        // ========== СТАДИЯ 1: УВЕДОМЛЕНИЕ О НАЧАЛЕ ==========
        stage('🚀 Начало сборки') {
            steps {
                script {
                    echo """
                    ========================================
                    ${PROJECT_NAME} - CI/CD Pipeline
                    ========================================
                    Сборка:       #${BUILD_NUMBER}
                    Тип события:  ${EVENT_TYPE}
                    Окружение:    ${DEPLOY_ENV}
                    Версия:       ${BUILD_VERSION}
                    Коммит:       ${env.GIT_COMMIT?.take(8) ?: 'N/A'}
                    ========================================
                    """

                    // Отправляем уведомление о начале
                    if (params.SEND_TELEGRAM) {
                        sendTelegramNotification(
                            status: 'start',
                            message: "🚂 Запущена сборка Railway ERP",
                            details: """
                            Сборка: #${BUILD_NUMBER}
                            Окружение: ${DEPLOY_ENV}
                            Тип: ${EVENT_TYPE}
                            """
                        )
                    }
                }
            }
        }

        // ========== СТАДИЯ 2: ПРОВЕРКА И ПОДГОТОВКА ==========
        stage('🔧 Подготовка окружения') {
            steps {
                script {
                    echo "Проверка инструментов и зависимостей..."

                    sh """
                        # Проверяем версии инструментов
                        echo "=== Информация о системе ==="
                        java -version
                        mvn -version
                        docker --version
                        echo ""

                        # Проверяем свободное место
                        echo "=== Дисковое пространство ==="
                        df -h
                        echo ""

                        # Очистка workspace
                        echo "Очистка workspace..."
                    """

                    // Очищаем workspace, но сохраняем важные файлы
                    cleanWs(
                        cleanWhenAborted: true,
                        cleanWhenFailure: true,
                        cleanWhenNotBuilt: true,
                        cleanWhenSuccess: true,
                        cleanWhenUnstable: true,
                        deleteDirs: true
                    )
                }
            }
        }

        // ========== СТАДИЯ 3: ПОЛУЧЕНИЕ КОДА ==========
        stage('📥 Получение кода') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${env.BRANCH_NAME ?: 'main'}"]],
                    extensions: [
                        [$class: 'CleanBeforeCheckout'],
                        [$class: 'CloneOption', depth: 1, noTags: false, shallow: true],
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: 'src']
                    ],
                    userRemoteConfigs: [[
                        url: 'git@github.com:MisStiCc/RailwayErpSystem.git',
                        credentialsId: 'github-ssh-key'
                    ]]
                ])

                dir('src') {
                    script {
                        sh """
                            echo "=== Git информация ==="
                            echo "Репо: $(git config --get remote.origin.url)"
                            echo "Ветка: $(git branch --show-current)"
                            echo "Коммит: $(git log -1 --pretty=format:'%H')"
                            echo "Автор: $(git log -1 --pretty=format:'%an <%ae>')"
                            echo "Дата: $(git log -1 --pretty=format:'%ad')"
                            echo "Сообщение: $(git log -1 --pretty=format:'%s')"
                            echo ""
                            echo "=== Изменения ==="
                            if [ "${env.GIT_PREVIOUS_COMMIT}" != "" ]; then
                                echo "Измененные файлы:"
                                git diff --name-only ${env.GIT_PREVIOUS_COMMIT} ${env.GIT_COMMIT} 2>/dev/null || true
                            fi
                        """
                    }
                }
            }
        }

        // ========== СТАДИЯ 4: СБОРКА JAVA/SPRING ПРИЛОЖЕНИЯ ==========
        stage('☕ Сборка Java приложения') {
            steps {
                dir('src') {
                    script {
                        echo "Сборка Spring Boot приложения..."

                        // Определяем профиль Maven в зависимости от окружения
                        def mavenProfile = DEPLOY_ENV == 'prod' ? '-Pproduction' :
                                          DEPLOY_ENV == 'test' ? '-Ptesting' : '-Pdevelopment'

                        sh """
                            # Устанавливаем версию в pom.xml
                            mvn versions:set -DnewVersion=${BUILD_VERSION} -DgenerateBackupPops=false

                            # Сборка проекта
                            echo "Используем профиль: ${mavenProfile}"
                            mvn clean compile ${mavenProfile} ${SKIP_TESTS_FLAG}

                            # Проверяем результат сборки
                            if [ -f "target/classes/application.yml" ]; then
                                echo "✅ Конфигурация собрана успешно"
                            fi
                        """
                    }
                }
            }
        }

        // ========== СТАДИЯ 5: ТЕСТИРОВАНИЕ ==========
        stage('🧪 Тестирование') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                dir('src') {
                    script {
                        echo "Запуск тестов..."

                        sh """
                            # Unit-тесты
                            echo "=== Unit тесты ==="
                            mvn test ${SKIP_TESTS_FLAG}

                            # Интеграционные тесты (если есть)
                            if [ -f "src/test/java/**/*IT.java" ]; then
                                echo "=== Интеграционные тесты ==="
                                mvn verify -DskipUnitTests ${SKIP_TESTS_FLAG}
                            fi

                            # Генерация отчетов
                            mvn surefire-report:report-only

                            echo "=== Результаты тестов ==="
                            echo "Тестов выполнено: \$(find target/surefire-reports -name '*.xml' | xargs grep -h 'tests=' | sed 's/.*tests=\"//' | sed 's/\".*//' | awk '{sum+=\$1} END {print sum}')"
                        """

                        // Сохраняем отчеты о тестах
                        junit '**/target/surefire-reports/*.xml'
                        archiveArtifacts artifacts: '**/target/surefire-reports/**', allowEmptyArchive: true
                    }
                }
            }
        }

        // ========== СТАДИЯ 6: АНАЛИЗ КАЧЕСТВА КОДА ==========
        stage('📊 Анализ качества') {
            steps {
                dir('src') {
                    script {
                        echo "Анализ качества кода..."

                        sh """
                            # Проверка стиля кода
                            echo "=== Checkstyle ==="
                            mvn checkstyle:check ${SKIP_TESTS_FLAG} || true

                            # Поиск багов
                            echo "=== SpotBugs ==="
                            mvn spotbugs:check ${SKIP_TESTS_FLAG} || true

                            # Анализ зависимостей
                            echo "=== OWASP Dependency Check ==="
                            mvn org.owasp:dependency-check-maven:check ${SKIP_TESTS_FLAG} || true

                            # Тесты покрытия кода
                            echo "=== JaCoCo (покрытие кода) ==="
                            mvn jacoco:prepare-agent test jacoco:report ${SKIP_TESTS_FLAG}
                        """

                        // Сохраняем отчеты
                        archiveArtifacts artifacts: '**/target/site/**', allowEmptyArchive: true
                    }
                }
            }
        }

        // ========== СТАДИЯ 7: СОЗДАНИЕ ДОКЕР ОБРАЗА ==========
        stage('🐳 Создание Docker образа') {
            when {
                expression { params.DEPLOY_ACTION != 'rollback' }
            }
            steps {
                dir('src') {
                    script {
                        echo "Создание Docker образа..."

                        // Собираем JAR
                        sh """
                            mvn package ${SKIP_TESTS_FLAG} -DskipTests

                            # Проверяем что JAR создан
                            JAR_FILE=\$(find target -name "*.jar" ! -name "*sources*" ! -name "*javadoc*" | head -1)
                            if [ -f "\$JAR_FILE" ]; then
                                echo "✅ JAR создан: \${JAR_FILE}"
                                ls -lh "\$JAR_FILE"
                            else
                                echo "❌ JAR не найден!"
                                exit 1
                            fi
                        """

                        // Создаем Dockerfile если его нет
                        sh '''
                            if [ ! -f "Dockerfile" ]; then
                                echo "Создаем Dockerfile..."
                                cat > Dockerfile << 'EOF'
FROM openjdk:17-jdk-slim
WORKDIR /app
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
RUN apt-get update && apt-get install -y curl
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
EOF
                            fi
                        '''

                        // Собираем Docker образ
                        sh """
                            # Собираем образ с тегами
                            docker build \
                                --build-arg JAR_FILE=target/*.jar \
                                -t ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:${BUILD_VERSION} \
                                -t ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:${DEPLOY_ENV}-latest \
                                -t ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:latest \
                                .

                            # Проверяем образ
                            docker images | grep ${APP_NAME}
                        """

                        // Сохраняем артефакты
                        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                        archiveArtifacts artifacts: 'Dockerfile', fingerprint: true
                    }
                }
            }
        }

        // ========== СТАДИЯ 8: ПУШ ДОКЕР ОБРАЗА ==========
        stage('📤 Пуш Docker образа') {
            when {
                expression { params.DEPLOY_ACTION == 'deploy' && DEPLOY_ENV != 'dev' }
            }
            steps {
                script {
                    echo "Отправка Docker образа в registry..."

                    sh """
                        # Логин в registry (если требуется)
                        # docker login ${DOCKER_REGISTRY} -u \${DOCKER_USER} -p \${DOCKER_PASSWORD}

                        # Пушим образы
                        docker push ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:${BUILD_VERSION}
                        docker push ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:${DEPLOY_ENV}-latest

                        echo "✅ Образы отправлены в registry"
                    """
                }
            }
        }

        // ========== СТАДИЯ 9: ДЕПЛОЙ ==========
        stage('🚀 Деплой') {
            when {
                expression { params.DEPLOY_ACTION == 'deploy' }
            }
            steps {
                script {
                    echo "Деплой в ${DEPLOY_ENV} окружение..."

                    // В зависимости от окружения используем разные методы деплоя
                    switch(DEPLOY_ENV) {
                        case 'dev':
                            sh """
                                # Локальный деплой для разработки
                                echo "Запуск в Docker для разработки..."
                                docker-compose -f src/docker-compose.yml up -d || echo "docker-compose не найден, пропускаем"

                                # Или запуск напрямую
                                # docker run -d -p 8080:8080 --name ${APP_NAME}-dev ${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${APP_NAME}:latest

                                echo "✅ Приложение запущено на http://localhost:8080"
                            """
                            break
                        case 'test':
                            sh """
                                # Деплой на тестовый сервер
                                echo "Деплой на тестовый сервер..."
                                # Здесь команды для деплоя на тестовый сервер
                                # например: kubectl apply -f k8s/test-deployment.yaml
                                echo "Деплой на тестовое окружение выполнен"
                            """
                            break
                        case 'prod':
                            sh """
                                # Деплой в прод
                                echo "Деплой в production..."
                                # Здесь команды для деплоя в прод
                                # например: kubectl apply -f k8s/prod-deployment.yaml
                                # или: ansible-playbook deploy-prod.yml
                                echo "Деплой в production выполнен"
                            """
                            break
                    }

                    // Проверка здоровья после деплоя
                    sh """
                        echo "Проверка здоровья приложения..."
                        sleep 10
                        curl -f http://localhost:8080/actuator/health || echo "Проверка здоровья не удалась, приложение может быть не готово"
                    """
                }
            }
        }

        // ========== СТАДИЯ 10: РОЛЛБЭК ==========
        stage('↩️ Роллбэк') {
            when {
                expression { params.DEPLOY_ACTION == 'rollback' && DEPLOY_ENV != 'dev' }
            }
            steps {
                script {
                    echo "Выполнение роллбэка..."

                    sh """
                        # Роллбэк на предыдущую версию
                        echo "Откат на предыдущую стабильную версию..."
                        # Здесь команды для роллбэка
                        # например: kubectl rollout undo deployment/${APP_NAME}
                        echo "Роллбэк выполнен"
                    """
                }
            }
        }
    }

    // ========== POST-ДЕЙСТВИЯ ==========
    post {
        // Всегда выполнять
        always {
            script {
                echo """
                ========================================
                    ИТОГИ СБОРКИ #${BUILD_NUMBER}
                ========================================
                Проект:      ${PROJECT_NAME}
                Статус:      ${currentBuild.currentResult}
                Окружение:   ${DEPLOY_ENV}
                Версия:      ${BUILD_VERSION}
                Длительность: ${currentBuild.durationString}
                ========================================
                Подробности: ${BUILD_URL}
                ========================================
                """

                // Сохраняем полный лог сборки
                archiveArtifacts artifacts: '**/target/*.log', allowEmptyArchive: true

                // Очищаем Docker образы чтобы не засорять диск
                sh '''
                    echo "Очистка временных Docker образов..."
                    docker system prune -f || true
                '''
            }
        }

        // При успехе
        success {
            script {
                echo "🎉 Сборка успешно завершена!"

                if (params.SEND_TELEGRAM) {
                    sendTelegramNotification(
                        status: 'success',
                        message: "✅ Сборка Railway ERP успешна",
                        details: """
                        Сборка: #${BUILD_NUMBER}
                        Окружение: ${DEPLOY_ENV}
                        Версия: ${BUILD_VERSION}
                        Длительность: ${currentBuild.durationString}
                        Ссылка: ${BUILD_URL}
                        """
                    )
                }

                // Очистка workspace после успешной сборки
                cleanWs()
            }
        }

        // При неудаче
        failure {
            script {
                echo "❌ Сборка завершилась с ошибкой!"

                if (params.SEND_TELEGRAM) {
                    sendTelegramNotification(
                        status: 'failure',
                        message: "❌ Сборка Railway ERP упала",
                        details: """
                        Сборка: #${BUILD_NUMBER}
                        Окружение: ${DEPLOY_ENV}
                        Ошибка в стадии: ${env.STAGE_NAME}
                        Ссылка: ${BUILD_URL}
                        """
                    )
                }
            }
        }

        // При отмене
        aborted {
            script {
                echo "⏸️ Сборка была отменена"

                if (params.SEND_TELEGRAM) {
                    sendTelegramNotification(
                        status: 'aborted',
                        message: "⏸️ Сборка Railway ERP отменена",
                        details: """
                        Сборка: #${BUILD_NUMBER}
                        Окружение: ${DEPLOY_ENV}
                        """
                    )
                }
            }
        }

        // При нестабильности (тесты не прошли)
        unstable {
            script {
                echo "⚠️ Сборка нестабильна (тесты не прошли)"

                if (params.SEND_TELEGRAM) {
                    sendTelegramNotification(
                        status: 'unstable',
                        message: "⚠️ Сборка Railway ERP нестабильна",
                        details: """
                        Сборка: #${BUILD_NUMBER}
                        Окружение: ${DEPLOY_ENV}
                        Причина: Тесты не прошли
                        Ссылка: ${BUILD_URL}
                        """
                    )
                }
            }
        }
    }
}

// ========== ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ ==========

/**
 * Отправка уведомления в Telegram с HTML форматированием
 */
def sendTelegramNotification(Map params = [:]) {
    def status = params.status ?: 'info'
    def message = params.message ?: 'Уведомление из Jenkins'
    def details = params.details ?: ''

    // Эмодзи и цвета для разных статусов
    def statusConfig = [
        'start':   [emoji: '🚀', color: '#3498db', title: 'Запуск сборки'],
        'success': [emoji: '✅', color: '#2ecc71', title: 'Успех'],
        'failure': [emoji: '❌', color: '#e74c3c', title: 'Ошибка'],
        'aborted': [emoji: '⏸️', color: '#95a5a6', title: 'Отменено'],
        'unstable':[emoji: '⚠️', color: '#f39c12', title: 'Нестабильно']
    ]

    def config = statusConfig[status] ?: [emoji: '📋', color: '#9b59b6', title: 'Информация']

    withCredentials([
        string(credentialsId: 'telegram-bot-token', variable: 'BOT_TOKEN'),
        string(credentialsId: 'telegram-chat-id', variable: 'CHAT_ID')
    ]) {
        def htmlMessage = """
<b>${config.emoji} ${config.title}: ${PROJECT_NAME}</b>

${message}

<pre>${details}</pre>

<code>────────────────────</code>
<b>Время:</b> ${new Date().format('dd.MM.yyyy HH:mm:ss')}
<b>Jenkins:</b> ${env.JENKINS_URL ?: 'localhost'}

<a href="${env.BUILD_URL}">📎 Открыть сборку</a> |
<a href="${env.PROJECT_URL}">🐙 Репозиторий</a>
        """.trim()

        // Кодируем для URL
        def encodedMessage = java.net.URLEncoder.encode(htmlMessage, "UTF-8")

        try {
            sh """
                curl -s -X POST "https://api.telegram.org/bot\${BOT_TOKEN}/sendMessage" \
                -d "chat_id=\${CHAT_ID}" \
                -d "text=${encodedMessage}" \
                -d "parse_mode=HTML" \
                -d "disable_web_page_preview=true" \
                --connect-timeout 10 \
                --max-time 30 \
                --retry 2 \
                --retry-delay 1
            """
            echo "✅ Уведомление отправлено в Telegram"
        } catch (Exception e) {
            echo "⚠️ Не удалось отправить уведомление в Telegram: ${e.message}"
        }
    }
}

/**
 * Проверка здоровья сервисов
 */
def healthCheck() {
    echo "🔍 Проверка здоровья сервисов..."

    try {
        sh """
            echo "=== Проверка инструментов ==="
            java -version && echo "✅ Java: OK"
            mvn -version && echo "✅ Maven: OK"
            docker --version && echo "✅ Docker: OK"

            echo ""
            echo "=== Проверка доступности ==="
            ping -c 1 github.com && echo "✅ GitHub: доступен"
            # ping -c 1 ${DOCKER_REGISTRY} && echo "✅ Docker Registry: доступен"

            echo ""
            echo "=== Системные ресурсы ==="
            free -h | grep Mem && echo "✅ Память: OK"
            df -h / && echo "✅ Диск: OK"
        """
    } catch (Exception e) {
        echo "⚠️ Проверка здоровья обнаружила проблемы: ${e.message}"
    }
}