pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: 'https://github.com/Miziday/CI-CD-AutoTests.git'
            }
        }
        
        stage('Start QEMU') {
            steps {
                script {
                    sh 'chmod +x qemu_start.sh'
                    sh 'Xvfb :99 -screen 0 1024x768x24 &'  
                    sh './qemu_start.sh &'
                    
                    def maxAttempts = 66
                    def waitTime = 3
                    def attempts = 0
                    def bmcAvailable = false
                    
                    echo "Ожидание доступности OpenBMC..."
                    
                    while (attempts < maxAttempts && !bmcAvailable) {
                        attempts++
                        try {
                            def status = sh(
                                script: 'curl -k -s -o /dev/null -w "%{http_code}" https://localhost:2443/redfish/v1',
                                returnStdout: true
                            ).trim()
                            
                            if (status == "200") {
                                bmcAvailable = true
                                echo "OpenBMC доступен после ${attempts * waitTime} секунд"
                            } else {
                                sleep(waitTime)
                            }
                        } catch (Exception e) {
                            sleep(waitTime)
                        }
                    }
                    
                    if (!bmcAvailable) {
                        error("OpenBMC не стал доступен после ${maxAttempts * waitTime} секунд ожидания")
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'romulus/*.mtd', allowEmptyArchive: true
                }
            }
        }


        stage('Check OpenBMC Availability') {
            steps {
                script {
                    sh '''
                    echo "Проверяем доступность OpenBMC..."
                    if curl -k https://localhost:2443; then
                        echo "OpenBMC доступен!"
                    else
                        echo "Ошибка: OpenBMC не отвечает на localhost:2443"
                        exit 1
                    fi
                    '''
                }
            }
        }
        
        stage('Run Auto Tests') {
            steps {
                dir('AuthTests') {
                    sh '/opt/venv/bin/pytest -v --junitxml=auth-results.xml openbmc_auth_tests.py'
                }
            }
            post {
                always {
                    junit 'AuthTests/auth-results.xml'
                    archiveArtifacts artifacts: 'AuthTests/auth-results.xml'
                }
            }
        }

        stage('Run WebUI Tests') {
            steps {
                dir('ApiTests') {
                    sh '/opt/venv/bin/pytest -v --junitxml=webui-results.xml test_redfish.py'
                }
            }
            post {
                always {
                    junit 'ApiTests/webui-results.xml'
                    archiveArtifacts artifacts: 'ApiTests/webui-results.xml'
                }
            }
        }
        
        stage('Run Load Tests') {
            steps {
                dir('LoadTests') {
                    script {
                        sh '''
                        /opt/venv/bin/locust -f locustfile.py \
                            --headless \
                            --users 300 \
                            --spawn-rate 10 \
                            --run-time 10s \
                            --host=https://localhost:2443 \
                            --html load_test_report.html \
                            --csv load_test \
                            --only-summary
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'LoadTests/load_test_report.html, LoadTests/load_test_*.csv'
                }
            }
        }
    }
}
