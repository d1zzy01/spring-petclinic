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

        stage('Wait for Production App') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                    if curl -fsS http://production-server:8080 >/dev/null; then
                        exit 0
                    fi
                    echo "Waiting for production-server:8080"
                    sleep 10
                    done
                    echo "App did not become ready in time"
                    docker exec production-server ps -ef || true
                    docker exec production-server bash -lc 'tail -n 200 /opt/petclinic/petclinic.log' || true
                    exit 1
                '''
            }
        }

        stage('OWASP ZAP Security Scan') {
            steps {
                sh '''
                    mkdir -p "$(pwd)/zap"
                    chmod 777 "$(pwd)/zap"

                    cleanup() {
                    docker rm -f zap-scan >/dev/null 2>&1 || true
                    }
                    trap cleanup EXIT
                    cleanup

                    docker run --name zap-scan \
                    --network devsecops-net \
                    --user root \
                    ghcr.io/zaproxy/zaproxy:stable \
                    bash -lc "mkdir -p /zap/wrk && /zap/zap-baseline.py -t http://production-server:8080 -r zap-report.html -I"

                    docker cp zap-scan:/zap/wrk/zap-report.html "$(pwd)/zap/zap-report.html"
                    test -s "$(pwd)/zap/zap-report.html"
                '''
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
