pipeline {
    agent any
    
    triggers {
        gitlab(
            triggerOnPush: true,
            triggerOnMergeRequest: true,
            branchFilterType: "All",
            secretToken: "your-secret-token"
        )
    }

    stages {
        stage('Build') {
            steps {
                echo 'Сборка запущена по MR'
                // ваши шаги сборки
            }
        }
    }
}