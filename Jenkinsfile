pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Setup Environment') {
            steps {
                echo '🔧 Setting up build environment...'
                sh '''
                    echo "=== Java Version ==="
                    java -version
                    echo ""
                    echo "=== Maven Version ==="
                    chmod +x mvnw 2>/dev/null || true
                    ./mvnw --version || echo "Maven wrapper not available"
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Build started for branch: ${env.BRANCH_NAME}"
                echo "📝 Commit: ${GIT_COMMIT}"
            }
        }

        // ==================== НОВЫЙ ЭТАП: КЭШИРОВАНИЕ Maven ====================
        stage('Maven Dependencies Cache') {
            steps {
                echo '📦 Setting up Maven dependencies cache...'
                script {
                    // Используем простой подход с общей папкой для кэша
                    sh '''
                        # Настройка путей для кэширования
                        MAVEN_CACHE_DIR="/var/jenkins_home/maven-cache"
                        LOCAL_MAVEN_REPO="${HOME}/.m2/repository"

                        echo "📁 Maven cache directory: $MAVEN_CACHE_DIR"
                        echo "📁 Local Maven repo: $LOCAL_MAVEN_REPO"

                        # Создаем директории если их нет
                        mkdir -p "$MAVEN_CACHE_DIR"
                        mkdir -p "$LOCAL_MAVEN_REPO"

                        # Проверяем, есть ли кэш зависимостей
                        if [ -d "$MAVEN_CACHE_DIR" ] && [ "$(ls -A $MAVEN_CACHE_DIR 2>/dev/null)" ]; then
                            echo "🔄 Restoring Maven dependencies from cache..."

                            # Копируем кэш в локальную папку Maven
                            # Используем rsync для эффективного копирования
                            if command -v rsync &> /dev/null; then
                                rsync -aq "$MAVEN_CACHE_DIR/" "$LOCAL_MAVEN_REPO/"
                            else
                                cp -r "$MAVEN_CACHE_DIR/"* "$LOCAL_MAVEN_REPO/" 2>/dev/null || true
                            fi

                            echo "✅ Maven cache restored successfully"
                            echo "📊 Cache size: $(du -sh $LOCAL_MAVEN_REPO 2>/dev/null | cut -f1) (approx)"
                        else
                            echo "📥 No cache found, will download dependencies..."
                            echo "⚠️ First build or cache cleared"
                        fi

                        # Скачиваем зависимости (используем go-offline для кэширования всех зависимостей)
                        echo "🔍 Resolving Maven dependencies..."
                        ./mvnw dependency:go-offline -DskipTests -q || {
                            echo "⚠️ Some dependencies may require network access during build"
                            echo "📥 Downloading essential dependencies..."
                            ./mvnw dependency:resolve -DskipTests -q
                        }

                        # Создаем симлинк для ускорения доступа
                        ln -sf "$LOCAL_MAVEN_REPO" "$WORKSPACE/.m2-repo-cache" 2>/dev/null || true
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Building Railway ERP System...'

                script {
                    if (fileExists('pom.xml')) {
                        echo '📦 Maven project detected'
                        sh '''
                            echo "=== Building with Maven (Java 17) ==="
                            echo "📊 Using cached dependencies from: ${HOME}/.m2/repository"

                            # Показываем размер кэша перед сборкой
                            du -sh ${HOME}/.m2/repository 2>/dev/null || echo "Cannot determine cache size"

                            ./mvnw clean compile -DskipTests \
                                -Djava.version=17 \
                                -Dmaven.compiler.source=17 \
                                -Dmaven.compiler.target=17 \
                                -Dmaven.repo.local=${HOME}/.m2/repository
                        '''
                    } else {
                        error '❌ pom.xml not found!'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Running tests...'

                script {
                    if (fileExists('pom.xml')) {
                        sh '''
                            echo "🧪 Running tests with cached dependencies..."
                            ./mvnw test -Djava.version=17 -Dmaven.repo.local=${HOME}/.m2/repository
                        '''
                        junit 'target/surefire-reports/**/*.xml'
                    }
                }
            }
        }

        stage('Package') {
            steps {
                echo '📦 Packaging application...'

                script {
                    if (fileExists('pom.xml')) {
                        sh '''
                            echo "📦 Creating package with cached dependencies..."
                            ./mvnw package -DskipTests \
                                -Djava.version=17 \
                                -Dmaven.repo.local=${HOME}/.m2/repository

                            # Показываем информацию о созданном артефакте
                            echo "✅ Package created:"
                            ls -la target/*.jar 2>/dev/null || echo "No JAR files found in target/"
                        '''
                    }
                }
            }
        }

        // ==================== НОВЫЙ ЭТАП: Сохранение кэша ====================
        stage('Save Maven Cache') {
            steps {
                echo '💾 Saving Maven dependencies to cache...'
                script {
                    sh '''
                        # Настройка путей
                        MAVEN_CACHE_DIR="/var/jenkins_home/maven-cache"
                        LOCAL_MAVEN_REPO="${HOME}/.m2/repository"

                        echo "💾 Saving dependencies to cache..."
                        echo "📁 Source: $LOCAL_MAVEN_REPO"
                        echo "📁 Destination: $MAVEN_CACHE_DIR"

                        # Очищаем старый кэш (оставляем только последний)
                        if [ -d "$MAVEN_CACHE_DIR" ]; then
                            echo "🧹 Cleaning old cache..."
                            rm -rf "${MAVEN_CACHE_DIR:?}/*"
                        fi

                        # Создаем директорию для кэша
                        mkdir -p "$MAVEN_CACHE_DIR"

                        # Копируем зависимости в кэш
                        echo "📤 Copying dependencies to cache..."
                        if command -v rsync &> /dev/null; then
                            rsync -aq "$LOCAL_MAVEN_REPO/" "$MAVEN_CACHE_DIR/"
                        else
                            cp -r "$LOCAL_MAVEN_REPO/"* "$MAVEN_CACHE_DIR/" 2>/dev/null || true
                        fi

                        # Проверяем что скопировалось
                        CACHE_SIZE=$(du -sh "$MAVEN_CACHE_DIR" 2>/dev/null | cut -f1)
                        echo "✅ Cache saved successfully!"
                        echo "📊 Cache size: $CACHE_SIZE (approx)"
                        echo "📈 Number of cached artifacts: $(find "$MAVEN_CACHE_DIR" -name "*.jar" -o -name "*.pom" 2>/dev/null | wc -l)"

                        # Создаем файл с метаданными кэша
                        echo "Creating cache metadata..."
                        cat > "$MAVEN_CACHE_DIR/cache-info.txt" << EOF
                        Cache created: $(date)
                        Build: ${JOB_NAME} #${BUILD_NUMBER}
                        Branch: ${BRANCH_NAME}
                        Commit: ${GIT_COMMIT}
                        Cache size: $CACHE_SIZE
                        Java version: $(java -version 2>&1 | head -1)
                        Maven version: $(./mvnw --version 2>/dev/null | grep "Apache Maven" || echo "Unknown")
                        EOF
                    '''
                }
            }
        }
    }

    post {
        always {
            echo '📊 Build completed'
            echo "Status: ${currentBuild.currentResult}"

            script {
                // Сохраняем логи сборки
                sh '''
                    mkdir -p build_logs
                    date > build_logs/build_time.txt
                    echo "Job: ${JOB_NAME} #${BUILD_NUMBER}" >> build_logs/build_time.txt
                    echo "Branch: ${BRANCH_NAME}" >> build_logs/build_time.txt
                    echo "Commit: ${GIT_COMMIT}" >> build_logs/build_time.txt
                    echo "Result: ${currentBuild.currentResult}" >> build_logs/build_time.txt

                    # Сохраняем информацию о кэше
                    echo "=== Maven Cache Info ===" >> build_logs/build_time.txt
                    du -sh ${HOME}/.m2/repository 2>/dev/null >> build_logs/build_time.txt || echo "Cache info unavailable" >> build_logs/build_time.txt
                '''

                // Архивируем артефакты
                archiveArtifacts artifacts: 'build_logs/**/*, target/*.jar', allowEmptyArchive: true

                // Сохраняем информацию о кэше в переменные сборки
                currentBuild.description = "Cache: ${currentBuild.currentResult}"
            }
        }

        success {
            echo '✅ Build successful!'
            script {
                // Отмечаем успешное сохранение кэша
                sh '''
                    echo "🎉 Build successful - cache preserved for next run"
                    echo "💾 Maven cache location: /var/jenkins_home/maven-cache"
                '''
            }
        }

        failure {
            echo '❌ Build failed!'
            script {
                // При неудачной сборке можно очистить кэш
                sh '''
                    echo "⚠️ Build failed - cache might be corrupted"
                    echo "💡 Consider clearing cache if builds continue to fail"

                    # Опционально: очистка кэша при постоянных ошибках
                    # if [ -d "/var/jenkins_home/maven-cache" ]; then
                    #     echo "🧹 Clearing potentially corrupted cache..."
                    #     rm -rf /var/jenkins_home/maven-cache/*
                    # fi
                '''
            }
        }

        cleanup {
            echo '🧹 Cleaning workspace (preserving Maven cache)...'
            script {
                // Очищаем workspace, но сохраняем .m2/repository для будущих сборок
                sh '''
                    echo "Preserving Maven cache at: ${HOME}/.m2/repository"
                    echo "Cleaning workspace except for build artifacts..."

                    # Удаляем временные файлы, но сохраняем важное
                    find . -name "target" -type d -exec rm -rf {} + 2>/dev/null || true
                    rm -rf build_logs 2>/dev/null || true

                    # Показываем что осталось
                    echo "Remaining in workspace:"
                    ls -la
                '''
            }
        }
    }

    environment {
        PROJECT_NAME = 'RailwayERP'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_URL = "${env.BUILD_URL}"
        // Добавляем переменные для кэширования
        MAVEN_CACHE_DIR = '/var/jenkins_home/maven-cache'
        MAVEN_OPTS = '-Dmaven.repo.local=${HOME}/.m2/repository -XX:+TieredCompilation -XX:TieredStopAtLevel=1'
    }

    options {
        // Дополнительные опции для лучшего кэширования
        skipDefaultCheckout(false)
        timeout(time: 30, unit: 'MINUTES')
        retry(1)
        timestamps()
        // Сохраняем workspace между сборками (опционально)
        preserveStashes(buildCount: 3)
    }
}