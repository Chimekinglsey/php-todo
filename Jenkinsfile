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
                    # Function to safely create directory if it doesn't exist
                    create_dir_if_not_exists() {
                        if [ ! -d "$1" ]; then
                            echo "Creating directory: $1"
                            mkdir -p "$1"
                        else
                            echo "Directory already exists: $1"
                        fi
                    }

                    # Create Laravel required directories
                    create_dir_if_not_exists "bootstrap/cache"
                    create_dir_if_not_exists "storage/framework/sessions"
                    create_dir_if_not_exists "storage/framework/views"
                    create_dir_if_not_exists "storage/framework/cache"
                    
                    # Set permissions regardless of whether directories were just created
                    echo "Setting directory permissions..."
                    chmod -R 775 storage bootstrap/cache || true
                    chown -R jenkins:jenkins storage bootstrap/cache || true
                    
                    # Copy environment file if it doesn't exist
                    if [ ! -f ".env" ]; then
                        echo "Copying .env file..."
                        cp .env.sample .env
                    else
                        echo ".env file already exists"
                    fi
                    
                    # Install composer if not present
                    if ! command -v composer &> /dev/null; then
                        echo "Installing composer..."
                        curl -sS https://getcomposer.org/installer | php
                        sudo mv composer.phar /usr/local/bin/composer
                    else
                        echo "Composer already installed"
                    fi
                '''
                
                // Run composer install with proper error handling
                sh '''
                    echo "Running composer install..."
                    composer install --no-interaction
                    if [ $? -eq 0 ]; then
                        echo "Composer install completed successfully"
                    else
                        echo "Composer install failed"
                        exit 1
                    fi
                '''
                
                // Run Laravel commands
                sh '''
                    echo "Generating application key..."
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