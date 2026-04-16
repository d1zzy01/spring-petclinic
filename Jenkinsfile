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

        stage('Burp Suite Security Scan') {
            steps {
                sh '''
                    docker run --rm \
                    --network devsecops-net \
                    -v $(pwd)/burp:/burp \
                    burpsuite \
                    http://production-server:8080 \
                    /burp/burp-report.html || true
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
                        reportName:            'Burp Suite Security Report'
                    ])
                }
            }
        }

        stage('Deploy to Production via Ansible') {
            when {
                branch 'main'   // only deploy from main branch
            }
            steps {
                sshagent(['production-ssh-key']) {   // Jenkins credential ID
                    sh '''
                        ansible-playbook \
                          -i ansible/inventory.ini \
                          ansible/deploy.yml \
                          --extra-vars "jar_path=$(ls $APP_JAR)"
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