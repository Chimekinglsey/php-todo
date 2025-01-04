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

        stage('Code Analysis') {
            steps {
                script {
                    // Check if phploc is installed
                    if (!fileExists('./vendor/bin/phploc')) {
                        echo 'phploc not found, installing...'
                        sh '''
                        if [ ! -f composer.json ]; then
                            echo "{}" > composer.json
                        fi
                        composer require --dev phploc/phploc:^3.0
                        '''
                    } else {
                        echo 'phploc already installed.'
                    }
                }

                // Run the phploc analysis
                sh './vendor/bin/phploc app/ --log-csv build/logs/phploc.csv'
            }
        }

        stage('Plot Metrics') {
            steps {
                plot(
                    csvFileName: 'build/logs/phploc.csv',
                    title: 'PHPLoc Metrics',
                    yaxis: 'Metrics',
                    group: 'Code Analysis',
                    style: 'line',
                )
            }
        }        
        
        stage ('Package Artifact') {
            steps {
                script {
                    // Create artifacts directory if it doesn't exist (with -p flag)
                    sh 'mkdir -p artifacts || true'
                    
                    // Remove existing archive if present
                    sh 'rm -f artifacts/php-todo.zip || true'
                    
                    // Package the application, excluding unnecessary files
                    sh '''
                        zip -r artifacts/php-todo.zip . \
                        -x "*.git*" \
                        -x "**/node_modules/**" \
                        -x "**/vendor/**" \
                        -x "*.zip" \
                        -x "**/storage/logs/*" \
                        -x "**/storage/framework/cache/*" \
                        -x "**/storage/framework/sessions/*"
                    '''
                }
            }
        }

        stage ('Upload Artifact to Artifactory') {
            steps {
                script {
                    try {
                        def server = Artifactory.server 'artifactory-server'
                        def buildInfo = Artifactory.newBuildInfo()
                        buildInfo.name = 'php-todo'
                        
                        // Get the build number from Jenkins
                        def buildNumber = env.BUILD_NUMBER
                        
                        // Create upload spec with simplified path
                        def uploadSpec = """{
                            "files": [
                                {
                                    "pattern": "artifacts/php-todo.zip",
                                    "target": "generic-local/php-todo/${buildNumber}/",
                                    "props": "build.name=php-todo;build.number=${buildNumber};type=zip;status=ready",
                                    "flat": true
                                }
                            ]
                        }"""

                        // Upload to Artifactory with retry mechanism
                        retry(3) {
                            server.upload spec: uploadSpec
                        }
                        
                        server.publishBuildInfo buildInfo
                    } catch (Exception e) {
                        echo "Failed to upload to Artifactory: ${e.message}"
                        currentBuild.result = 'UNSTABLE'
                        // Continue with deployment even if upload fails
                    }
                }
            }
        }

        stage ('Deploy to Dev Environment') {
            steps {
                build(
                    job: 'ansible-project-demo',
                    parameters: [string(name: 'env', value: 'dev')],
                    propagate: false,
                    wait: true
                )
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