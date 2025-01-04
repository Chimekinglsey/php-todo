pipeline {
    agent any

    environment {
        PHP_VERSION = '7.0'
    }

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
                git branch: 'main', url: 'https://github.com/Chimekinglsey/php-todo'
                sh 'git status'
                sh 'git log --oneline -5'
            }
        }

        stage('Setup PHP') {
            steps {
                sh '''
                    if ! command -v php7.0 &> /dev/null; then
                        sudo apt-get update
                        sudo apt-get install -y software-properties-common
                        sudo add-apt-repository -y ppa:ondrej/php
                        sudo apt-get update
                        sudo apt-get install -y php7.0 php7.0-cli php7.0-common php7.0-json php7.0-opcache php7.0-mysql php7.0-mbstring php7.0-mcrypt php7.0-zip php7.0-fpm php7.0-xml
                    fi
                '''
                sh 'sudo update-alternatives --set php /usr/bin/php7.0'
                sh 'php -v'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh 'mv .env.sample .env'
                sh '''
                    if ! command -v composer &> /dev/null; then
                        curl -sS https://getcomposer.org/installer | php
                        sudo mv composer.phar /usr/local/bin/composer
                    fi
                '''
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