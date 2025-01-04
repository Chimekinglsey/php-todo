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
                script {
                    def currentPhpVersion = sh(script: 'php -r "echo PHP_MAJOR_VERSION;"', returnStdout: true).trim()
                    if (currentPhpVersion == '7') {
                        echo "PHP 7.x is already installed. Skipping PHP setup."
                    } else {
                        sh '''
                            sudo apt-get update
                            sudo apt-get install -y software-properties-common
                            sudo add-apt-repository -y ppa:ondrej/php
                            sudo apt-get update
                            sudo apt-get install -y php7.0 php7.0-cli php7.0-common php7.0-json php7.0-opcache php7.0-mysql php7.0-mbstring php7.0-mcrypt php7.0-zip php7.0-fpm php7.0-xml
                            sudo update-alternatives --set php /usr/bin/php7.0
                        '''
                    }
                    sh 'php -v'
                }
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh '''
                    # Create directories and set permissions
                    create_dir_if_not_exists() {
                        if [ ! -d "$1" ]; then
                            echo "Creating directory: $1"
                            mkdir -p "$1"
                        else
                            echo "Directory already exists: $1"
                        fi
                    }

                    create_dir_if_not_exists "bootstrap/cache"
                    create_dir_if_not_exists "storage/framework/sessions"
                    create_dir_if_not_exists "storage/framework/views"
                    create_dir_if_not_exists "storage/framework/cache"
                    
                    chmod -R 775 storage bootstrap/cache || true
                    chown -R jenkins:jenkins storage bootstrap/cache || true
                    
                    if [ ! -f ".env" ]; then
                        cp .env.sample .env
                    fi
                    
                    if ! command -v composer &> /dev/null; then
                        curl -sS https://getcomposer.org/installer | php
                        sudo mv composer.phar /usr/local/bin/composer
                    fi
                '''
                
                sh '''
                    composer install --no-interaction
                '''
                
                // Test database connection before proceeding
                sh '''
                    echo "Testing database connection..."
                    # Install mysql-client if not present
                    if ! command -v mysql &> /dev/null; then
                        sudo apt-get update
                        sudo apt-get install -y mysql-client
                    fi
                    
                    # Test MySQL connection
                    if mysql -h 172.31.10.129 -u kings -plesson123 -e "use php_todo"; then
                        echo "Database connection successful!"
                    else
                        echo "Failed to connect to database"
                        exit 1
                    fi
                '''
                
                sh '''
                    php artisan config:clear
                    php artisan key:generate
                    echo "Running migrations..."
                    php artisan migrate --force
                    echo "Seeding database..."
                    php artisan db:seed --force
                '''
            }
        }

        stage('Execute Unit Tests') {
            steps {
                sh './vendor/bin/phpunit'
            } 
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo 'Pipeline failed! Check the logs for details.'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}