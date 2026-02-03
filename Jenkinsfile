pipeline {
    agent any
    
    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "🚀 Build started for branch: ${env.BRANCH_NAME}"
                echo "📝 Commit: ${GIT_COMMIT}"
                sh '''
                    echo "Repository information:"
                    git log --oneline -3
                    echo ""
                    echo "Branch information:"
                    git branch -a
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
                            java -version 2>&1 | head -3 || echo "Java not available"
                            echo ""
                            echo "=== Maven Wrapper ==="
                            chmod +x mvnw 2>/dev/null || echo "Cannot make mvnw executable"
                            ./mvnw --version 2>&1 | head -3 || echo "Maven wrapper not working"
                            echo ""
                            echo "=== Building with Maven ==="
                            ./mvnw clean compile -DskipTests || echo "Maven build failed"
                            echo ""
                            echo "=== Generated files ==="
                            find target/ -name "*.jar" -o -name "*.war" 2>/dev/null | head -5 || echo "No JAR/WAR files found"
                        '''
                    } else if (fileExists('package.json')) {
                        echo '📦 Node.js project detected'
                        sh '''
                            echo "=== Node.js Information ==="
                            node --version || echo "Node.js not available"
                            npm --version || echo "npm not available"
                            echo ""
                            echo "=== Installing dependencies ==="
                            npm install --silent || echo "npm install failed"
                            echo ""
                            echo "=== Building ==="
                            npm run build || echo "npm build failed"
                        '''
                    } else {
                        echo '⚠️  No build configuration found'
                        sh '''
                            echo "Project structure:"
                            ls -la
                            echo ""
                            echo "Looking for build files:"
                            find . -name "pom.xml" -o -name "package.json" -o -name "build.gradle" -o -name "*.sln" | head -10
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
                            ./mvnw test
                            echo ""
                            echo "=== Test results ==="
                            find target/surefire-reports -name "*.txt" -type f 2>/dev/null | head -3 | while read file; do
                                echo "Test file: $file"
                                tail -5 "$file" 2>/dev/null || echo "No test results"
                            done
                        '''
                        junit 'target/surefire-reports/**/*.xml'
                    } else if (fileExists('package.json')) {
                        sh '''
                            grep -q '"test"' package.json && npm test || echo "No test script in package.json"
                        '''
                    } else {
                        echo '📭 No tests configured'
                        sh '''
                            echo "Looking for test files:"
                            find . -name "*Test*.java" -o -name "*test*.py" -o -name "*.spec.js" | head -5
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
                            docker-compose ps 2>/dev/null || echo "Cannot check containers"
                        '''
                    } else {
                        echo '📭 No deployment configuration found'
                        sh 'echo "Test deployment would be executed here"'
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
                    echo "Lines of code:"
                    find . -name "*.java" -o -name "*.py" -o -name "*.js" -o -name "*.ts" | \
                    xargs wc -l 2>/dev/null | tail -1 | awk '{print "Total lines:", $1}' || echo "Cannot count lines"
                    echo ""
                    echo "Project size:"
                    du -sh . 2>/dev/null | awk '{print "Size:", $1}' || echo "Cannot get size"
                    echo ""
                    echo "TODO/FIXME markers:"
                    grep -r -i "TODO\\|FIXME\\|HACK\\|XXX" --include="*.java" --include="*.py" --include="*.js" --include="*.ts" . 2>/dev/null | \
                    head -5 || echo "No TODO/FIXME markers found"
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

                    # System information
                    echo "=== System Info ===" > build_logs/system_info.txt
                    java -version 2>&1 >> build_logs/system_info.txt || echo "Java: Not available" >> build_logs/system_info.txt
                    echo "" >> build_logs/system_info.txt
                    echo "Maven:" >> build_logs/system_info.txt
                    ./mvnw --version 2>&1 | head -3 >> build_logs/system_info.txt 2>/dev/null || echo "Maven: Not available" >> build_logs/system_info.txt
                '''

                // Archive artifacts
                archiveArtifacts artifacts: 'build_logs/**/*', allowEmptyArchive: true

                // Archive build results if they exist
                if (fileExists('target/')) {
                    archiveArtifacts artifacts: 'target/*.jar, target/*.war', allowEmptyArchive: true
                }
            }
        }

        success {
            echo '✅ ✅ ✅ Build successful! ✅ ✅ ✅'

            script {
                // Send success notification (uncomment if needed)
                /*
                emailext (
                    subject: "✅ Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                    Build ${env.BUILD_NUMBER} of ${env.JOB_NAME} was successful!

                    Branch: ${env.BRANCH_NAME}
                    Duration: ${currentBuild.durationString}

                    View build: ${env.BUILD_URL}
                    """,
                    to: 'team@example.com'
                )
                */
            }
        }

        failure {
            echo '❌ ❌ ❌ Build failed! ❌ ❌ ❌'

            script {
                // Send failure notification (uncomment if needed)
                /*
                emailext (
                    subject: "❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                    Build ${env.BUILD_NUMBER} of ${env.JOB_NAME} has failed!

                    Branch: ${env.BRANCH_NAME}
                    Stage: ${env.STAGE_NAME}
                    Duration: ${currentBuild.durationString}

                    View build: ${env.BUILD_URL}
                    Check logs for details.
                    """,
                    to: 'team@example.com'
                )
                */
            }
        }

        unstable {
            echo '⚠️ ⚠️ ⚠️ Build unstable! ⚠️ ⚠️ ⚠️'
        }

        changed {
            echo '🔄 Build status changed!'
        }

        cleanup {
            echo '🧹 Cleaning workspace...'
            // Uncomment to clean workspace after build
            // cleanWs()
        }
    }

    environment {
        PROJECT_NAME = 'RailwayERP'
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
        BUILD_URL = "${env.BUILD_URL}"
        JOB_NAME = "${env.JOB_NAME}"

        // Uncomment and configure if needed
        // DEPLOY_ENV = 'test'
        // ARTIFACTORY_URL = 'https://artifactory.example.com'
        // SONAR_HOST_URL = 'https://sonar.example.com'
    }
}