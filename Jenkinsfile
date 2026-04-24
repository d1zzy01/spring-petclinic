pipeline {
    agent any

    environment {
        SONAR_HOST_URL    = 'http://sonarqube:9000'
        SONAR_TOKEN       = credentials('sonar-token')
        APP_JAR           = 'target/spring-petclinic-*.jar'
        PRODUCTION_HOST   = 'production-server'       // ← change to your VM IP
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        // Allow Testcontainers to reach Docker-started containers from inside Jenkins container
        DOCKER_HOST                           = 'unix:///var/run/docker.sock'
        TESTCONTAINERS_HOST_OVERRIDE          = 'host.docker.internal'
        TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE = '/var/run/docker.sock'
    }

    triggers {
        // Poll SCM every 5 minutes for new commits
        pollSCM('* * * * *')
    }

    tools {
        maven 'Maven-3.9'   // must match the name set in Jenkins → Global Tools
        jdk   'JDK-17'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Unit Tests') {
            steps {
                sh 'mvn clean package -Dsurefire.excludes="**/*IntegrationTests.java"'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {   // name must match Jenkins SQ config
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=spring-petclinic \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy to Production via Ansible') {
            steps {
                sshagent(['production-ssh-key']) {
                    sh '''
                        ansible-playbook \
                        -i ansible/inventory.ini \
                        ansible/deploy.yml \
                        --extra-vars "jar_path=$(pwd)/target/spring-petclinic-4.0.0-SNAPSHOT.jar"
                    '''
                }
            }
        }

        stage('OWASP ZAP Security Scan') {
            steps {
                sh '''
                    mkdir -p $(pwd)/zap
                    chmod 777 $(pwd)/zap

                    # Remove any previous container
                    docker rm -f zap-scan 2>/dev/null || true

                    # Run ZAP with volume mount AND named container
                    docker run --name zap-scan \
                    --network devsecops-net \
                    -v $(pwd)/zap:/zap/wrk:rw \
                    --user root \
                    ghcr.io/zaproxy/zaproxy:stable \
                    bash -c "chmod 777 /zap/wrk && zap-baseline.py -t http://production-server:8080 -r zap-report.html -I || true" || true

                    # Copy report out just in case volume didnt work
                    docker cp zap-scan:/zap/wrk/zap-report.html $(pwd)/zap/zap-report.html 2>/dev/null || true

                    docker rm zap-scan 2>/dev/null || true

                    ls -lh $(pwd)/zap/ || true
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing:          false,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'burp',
                        reportFiles:           'burp-report.html',
                        reportName:            'OWASP ZAP Security Report'
                    ])
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — ${env.BUILD_URL}"
        }
        failure {
            echo "Pipeline FAILED — check logs at ${env.BUILD_URL}"
        }
    }
}