pipeline {
    agent any

    stages {

        stage("Initial cleanup") {
            steps {
                dir("${WORKSPACE}") {
                deleteDir()
                }
            }
        }
    
        stage('Checkout SCM') {
            steps {
                    cleanWs()
                    git branch: 'main', url: 'https://github.com/Chimekinglsey/php-todo'
                    sh 'git status'
                    sh 'git log --oneline -5'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                    sh 'mv .env.sample .env'
                    sh 'composer install'
                    sh 'php artisan migrate'
                    sh 'php artisan db:seed'
                    sh 'php artisan key:generate'
            }
        }

        stage('Execute Unit Tests') {
            steps {
                    sh './vendor/bin/phpunit'
            } 
        }
    }
}