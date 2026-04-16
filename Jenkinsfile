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
                // Start app in background for DAST scan
                sh 'java -jar $APP_JAR --server.port=8090 &'
                sh 'sleep 20'   // wait for app to start

                sh '''
                    docker run --rm \
                      --network devsecops-net \
                      -v $(pwd)/burp:/burp \
                      burpsuite-headless \
                      --unpause-spider-and-scanner \
                      --config-file=/burp/burp-config.json \
                      --project-file=/burp/petclinic-scan.burp \
                      --report-output=/burp/burp-report.html
                '''
            }
            post {
                always {
                    // Kill background app
                    sh "pkill -f 'spring-petclinic' || true"
                    // Publish HTML report
                    publishHTML(target: [
                        allowMissing:         false,
                        alwaysLinkToLastBuild: true,
                        keepAll:              true,
                        reportDir:            'burp',
                        reportFiles:          'burp-report.html',
                        reportName:           'Burp Suite Security Report'
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