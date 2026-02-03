pipeline {
    agent any
    
    tools {
        // Указываем Java 17 для сборки
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
                    ./mvnw --version || echo "Maven wrapper not available"
                    echo ""
                    echo "=== System Information ==="
                    uname -a
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Build started for branch: ${env.BRANCH_NAME}"
                echo "📝 Commit: ${GIT_COMMIT}"
                sh '''
                    echo "Repository information:"
                    git log --oneline -3
                '''
            }
        }

        stage('Build') {
            steps {
                echo '🔨 Building Railway ERP System...'

                script {
                    // Check for Docker Compose
                    if (fileExists('compose.yaml') || fileExists('docker-compose.yml')) {
                        echo '🐳 Found Docker Compose configuration'
                        sh '''
                            echo "--- Docker Compose ---"
                            if [ -f "compose.yaml" ]; then
                                head -15 compose.yaml
                            elif [ -f "docker-compose.yml" ]; then
                                head -15 docker-compose.yml
                            fi
                        '''
                    }

                    // Check for Maven project
                    if (fileExists('pom.xml')) {
                        echo '📦 Maven project detected'
                        sh '''
                            echo "=== Java Information ==="
                            java -version
                            echo ""
                            echo "=== Maven Wrapper ==="
                            chmod +x mvnw 2>/dev/null && echo "Maven wrapper permissions set"
                            ./mvnw --version 2>&1 | head -3
                            echo ""
                            echo "=== Building with Maven (Java 17) ==="
                            # Явно указываем Java 17 для компиляции
                            ./mvnw clean compile -DskipTests \
                                -Djava.version=17 \
                                -Dmaven.compiler.source=17 \
                                -Dmaven.compiler.target=17
                            echo ""
                            echo "=== Generated files ==="
                            find target/ -name "*.jar" -o -name "*.war" 2>/dev/null | head -5 || echo "No JAR/WAR files found"
                        '''
                    } else {
                        echo '⚠️  No Maven project found'
                    }
                }
            }

            post {
                success {
                    echo '✅ Build completed successfully'
                }
                failure {
                    echo '❌ Build failed'
                    script {
                        // Попробуем альтернативный способ сборки
                        echo 'Trying alternative build approach...'
                        sh '''
                            # Проверяем версию Java в pom.xml
                            echo "Checking Java version in pom.xml..."
                            grep -i "java.version" pom.xml

                            # Пробуем собрать с явными параметрами
                            ./mvnw clean compile -DskipTests \
                                -Dmaven.compiler.fork=true \
                                -Dmaven.compiler.executable=$(which java) \
                                -Dmaven.compiler.source=17 \
                                -Dmaven.compiler.target=17
                        '''
                    }
                }
            }
        }

        stage('Test') {
            steps {
                echo '🧪 Running tests...'

                script {
                    if (fileExists('pom.xml')) {
                        echo '📦 Running Maven tests'
                        sh '''
                            echo "=== Running Maven tests ==="
                            ./mvnw test -Djava.version=17
                            echo ""
                            echo "=== Test results ==="
                            find target/surefire-reports -name "*.txt" -type f 2>/dev/null | head -3 | while read file; do
                                echo "Test file: $file"
                                tail -5 "$file" 2>/dev/null || echo "No test results"
                            done
                        '''
                        junit 'target/surefire-reports/**/*.xml'
                    } else {
                        echo '📭 No tests configured'
                    }
                }
            }
        }

        stage('Package Application') {
            steps {
                echo '📦 Packaging application...'

                script {
                    if (fileExists('pom.xml')) {
                        sh '''
                            echo "=== Creating JAR package ==="
                            ./mvnw package -DskipTests \
                                -Djava.version=17 \
                                -Dmaven.compiler.source=17 \
                                -Dmaven.compiler.target=17
                            echo ""
                            echo "=== Created artifacts ==="
                            ls -la target/*.jar 2>/dev/null || echo "No JAR files created"
                        '''
                    }
                }
            }
        }

        stage('Deploy to Test') {
            when {
                branch 'main'
            }
            steps {
                echo '🚚 Deploying to test environment...'

                script {
                    if (fileExists('docker-compose.yml') || fileExists('compose.yaml')) {
                        echo '🐳 Deploying with Docker Compose'
                        sh '''
                            echo "Starting Docker Compose for testing..."
                            docker-compose up -d --build || echo "Docker Compose not available"
                            echo ""
                            echo "Checking containers:"
                            docker ps --filter "name=railway" || echo "No containers found"
                        '''
                    } else {
                        echo '📭 No deployment configuration found'
                    }
                }
            }
        }

        stage('Quality Check') {
            steps {
                echo '📊 Running quality checks...'

                sh '''
                    echo "=== Code Statistics ==="
                    echo ""
                    echo "Lines of Java code:"
                    find src/ -name "*.java" | xargs wc -l 2>/dev/null | tail -1 || echo "No Java files found"
                    echo ""
                    echo "Project structure:"
                    ls -la
                    echo ""
                    echo "TODO/FIXME markers:"
                    grep -r -i "TODO\\|FIXME\\|HACK\\|XXX" src/ 2>/dev/null | head -5 || echo "No TODO/FIXME markers found"
                '''
            }
        }
    }

    post {
        always {
            echo '📊 ======== Build completed ========'
            echo "Status: ${currentBuild.currentResult}"
            echo "Duration: ${currentBuild.durationString}"
            echo "URL: ${env.BUILD_URL}"

            script {
                // Create build logs
                sh '''
                    echo "Creating build logs..."
                    mkdir -p build_logs
                    date > build_logs/build_time.txt
                    echo "Job: ${JOB_NAME} #${BUILD_NUMBER}" >> build_logs/build_time.txt
                    echo "Branch: ${BRANCH_NAME}" >> build_logs/build_time.txt
                    echo "Commit: ${GIT_COMMIT}" >> build_logs/build_time.txt
                    echo "Result: ${currentBuild.currentResult}" > build_logs/result.txt
                    echo "Duration: ${currentBuild.durationString}" >> build_logs/result.txt

                    # Java information
                    echo "=== Java Info ===" >> build_logs/system_info.txt
                    java -version 2>&1 >> build_logs/system_info.txt

                    # Maven information
                    echo "" >> build_logs/system_info.txt
                    echo "=== Maven Info ===" >> build_logs/system_info.txt
                    ./mvnw --version 2>&1 | head -3 >> build_logs/system_info.txt 2>/dev/null || echo "Maven: Not available" >> build_logs/system_info.txt

                    # Build artifacts
                    echo "" >> build_logs/system_info.txt
                    echo "=== Build Artifacts ===" >> build_logs/system_info.txt
                    find target/ -name "*.jar" -type f 2>/dev/null >> build_logs/system_info.txt || echo "No artifacts" >> build_logs/system_info.txt
                '''

                // Archive artifacts
                archiveArtifacts artifacts: 'build_logs/**/*, target/*.jar', allowEmptyArchive: true
            }
        }

        success {
            echo '✅ ✅ ✅ Build successful! ✅ ✅ ✅'

            script {
                // Можно добавить Telegram уведомление здесь
                // sendTelegram("✅ Build successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
            }
        }

        failure {
            echo '❌ ❌ ❌ Build failed! ❌ ❌ ❌'

            script {
                // Можно добавить Telegram уведомление об ошибке
                // sendTelegram("❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
            }
        }

        cleanup {
            echo '🧹 Cleaning workspace...'
            // Раскомментировать для очистки workspace после сборки
            // cleanWs()
        }
    }

    environment {
        PROJECT_NAME = 'RailwayERP'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_URL = "${env.BUILD_URL}"
        JOB_NAME = "${env.JOB_NAME}"
        JAVA_HOME = "${tool 'jdk17'}"
    }
}