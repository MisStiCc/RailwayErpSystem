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

        stage('Build') {
            steps {
                echo '🔨 Building Railway ERP System...'

                script {
                    if (fileExists('pom.xml')) {
                        echo '📦 Maven project detected'
                        sh '''
                            echo "=== Building with Maven (Java 17) ==="
                            chmod +x mvnw 2>/dev/null
                            ./mvnw clean compile -DskipTests \
                                -Djava.version=17 \
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
                        sh './mvnw test -Djava.version=17'
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
                        sh './mvnw package -DskipTests -Djava.version=17'
                    }
                }
            }
        }
    }

    post {
        always {
            echo '📊 Build completed'
            echo "Status: ${currentBuild.currentResult}"

            script {
                sh '''
                    mkdir -p build_logs
                    date > build_logs/build_time.txt
                    echo "Job: ${JOB_NAME} #${BUILD_NUMBER}" >> build_logs/build_time.txt
                '''
                archiveArtifacts artifacts: 'build_logs/**/*, target/*.jar', allowEmptyArchive: true
            }
        }

        success {
            echo '✅ Build successful!'
        }

        failure {
            echo '❌ Build failed!'
        }

        cleanup {
            echo '🧹 Cleaning workspace...'
        }
    }

    environment {
        PROJECT_NAME = 'RailwayERP'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_URL = "${env.BUILD_URL}"
    }
}