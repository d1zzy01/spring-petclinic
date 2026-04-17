pipeline {
    agent any

    environment {
        SONAR_HOST_URL    = 'http://sonarqube:9000'
        SONAR_TOKEN       = credentials('sonar-token')
        APP_JAR           = 'target/spring-petclinic-*.jar'
        PRODUCTION_HOST   = 'production-server'       // ← change to your VM IP
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    triggers {
        // Poll SCM every 5 minutes for new commits
        pollSCM('H/5 * * * *')
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
                sh 'mvn clean package -DskipTests=false'
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

        stage('OWASP ZAP Security Scan') {
            steps {
                sh '''
                    mkdir -p $(pwd)/burp
                    chmod 777 $(pwd)/burp

                    docker run --rm \
                    --network devsecops_devsecops-net \
                    -v $(pwd)/burp:/zap/wrk/:rw \
                    --user root \
                    ghcr.io/zaproxy/zaproxy:stable \
                    bash -c "zap-baseline.py -t http://production-server:8080 -r burp-report.html -I || true; find / -name 'burp-report.html' 2>/dev/null; ls -la /zap/wrk/" || true

                    ls -lh $(pwd)/burp/ || true
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing:          true,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'burp',
                        reportFiles:           'burp-report.html',
                        reportName:            'OWASP ZAP Security Report'
                    ])
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