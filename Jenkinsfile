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
                    # Даем права на mvnw
                    chmod +x mvnw 2>/dev/null || true
                    ./mvnw --version || echo "Maven wrapper not available"
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Build started for branch: main"
                echo "📝 Commit: ${GIT_COMMIT}"
            }
        }

        stage('Maven Dependencies Cache') {
            steps {
                echo '📦 Setting up Maven dependencies cache...'
                sh '''
                    # Настройка путей для кэширования
                    MAVEN_CACHE_DIR="/var/jenkins_home/maven-cache"
                    LOCAL_MAVEN_REPO="${HOME}/.m2/repository"

                    echo "📁 Maven cache directory: $MAVEN_CACHE_DIR"
                    echo "📁 Local Maven repo: $LOCAL_MAVEN_REPO"

                    # Даем права на mvnw
                    chmod +x mvnw 2>/dev/null || true

                    # Создаем директории если их нет
                    mkdir -p "$MAVEN_CACHE_DIR"
                    mkdir -p "$LOCAL_MAVEN_REPO"

                    # Проверяем, есть ли кэш зависимостей
                    if [ -d "$MAVEN_CACHE_DIR" ] && [ "$(ls -A $MAVEN_CACHE_DIR 2>/dev/null)" ]; then
                        echo "🔄 Restoring Maven dependencies from cache..."

                        # Копируем кэш в локальную папку Maven
                        cp -r "$MAVEN_CACHE_DIR/"* "$LOCAL_MAVEN_REPO/" 2>/dev/null || true

                        echo "✅ Maven cache restored successfully"
                        DU_OUTPUT=$(du -sh $LOCAL_MAVEN_REPO 2>/dev/null | cut -f1)
                        echo "📊 Cache size: $DU_OUTPUT"
                    else
                        echo "📥 No cache found, will download dependencies..."
                        echo "⚠️ First build or cache cleared"
                    fi

                    # Скачиваем зависимости (используем go-offline для кэширования всех зависимостей)
                    echo "🔍 Resolving Maven dependencies..."
                    ./mvnw dependency:go-offline -DskipTests || {
                        echo "⚠️ Some dependencies may require network access during build"
                        echo "📥 Downloading essential dependencies..."
                        ./mvnw dependency:resolve -DskipTests
                    }
                '''
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
                            # Даем права на mvnw
                            chmod +x mvnw 2>/dev/null || true

                            # Показываем размер кэша перед сборкой
                            DU_OUTPUT=$(du -sh ${HOME}/.m2/repository 2>/dev/null | cut -f1)
                            echo "📊 Using cached dependencies, size: $DU_OUTPUT"

                            ./mvnw clean compile -DskipTests \
                                -Djava.version=17 \
                                -Dmaven.compiler.source=17 \
                                -Dmaven.compiler.target=17
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
                            chmod +x mvnw 2>/dev/null || true
                            ./mvnw test -Djava.version=17
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
                            chmod +x mvnw 2>/dev/null || true
                            ./mvnw package -DskipTests -Djava.version=17

                            # Показываем информацию о созданном артефакте
                            echo "✅ Package created:"
                            ls -la target/*.jar 2>/dev/null || echo "No JAR files found in target/"
                        '''
                    }
                }
            }
        }

        stage('Save Maven Cache') {
            steps {
                echo '💾 Saving Maven dependencies to cache...'
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
                        rm -rf "$MAVEN_CACHE_DIR"/*
                    fi

                    # Создаем директорию для кэша
                    mkdir -p "$MAVEN_CACHE_DIR"

                    # Копируем зависимости в кэш
                    echo "📤 Copying dependencies to cache..."
                    cp -r "$LOCAL_MAVEN_REPO/"* "$MAVEN_CACHE_DIR/" 2>/dev/null || true

                    # Проверяем что скопировалось
                    CACHE_SIZE=$(du -sh "$MAVEN_CACHE_DIR" 2>/dev/null | cut -f1)
                    echo "✅ Cache saved successfully!"
                    echo "📊 Cache size: $CACHE_SIZE"
                '''
            }
        }
    }

post {
    always {
        echo '📊 Build completed'
        echo "Status: ${currentBuild.currentResult}"

        script {
            // Сохраняем логи сборки
            sh """
                mkdir -p build_logs
                date > build_logs/build_time.txt
                echo "Job: ${JOB_NAME} #${BUILD_NUMBER}" >> build_logs/build_time.txt
                echo "Branch: main" >> build_logs/build_time.txt
                echo "Commit: ${GIT_COMMIT}" >> build_logs/build_time.txt
                echo "Result: ${currentBuild.currentResult}" >> build_logs/build_time.txt

                # Сохраняем информацию о кэше
                echo "=== Maven Cache Info ===" >> build_logs/build_time.txt
                du -sh \${HOME}/.m2/repository 2>/dev/null >> build_logs/build_time.txt || echo "Cache info unavailable" >> build_logs/build_time.txt
            """

            // Архивируем артефакты
            archiveArtifacts artifacts: 'build_logs/**/*, target/*.jar', allowEmptyArchive: true
        }
    }

        success {
            echo '✅ Build successful!'
            script {
                sh '''
                    echo "🎉 Build successful - cache preserved for next run"
                    echo "💾 Maven cache location: /var/jenkins_home/maven-cache"
                '''
            }
        }

        failure {
            echo '❌ Build failed!'
            script {
                sh '''
                    echo "⚠️ Build failed - cache might be corrupted"
                    echo "💡 Consider clearing cache if builds continue to fail"
                '''
            }
        }

        cleanup {
            echo '🧹 Cleaning workspace (preserving Maven cache)...'
            script {
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
    }

    options {
        skipDefaultCheckout(false)
        timeout(time: 30, unit: 'MINUTES')
        retry(1)
        timestamps()
    }
}