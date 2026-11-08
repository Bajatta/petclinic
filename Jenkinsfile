pipeline {
    agent any
    environment {
        NEXUS_USER     = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO     = credentials('nexus-repo')   // <nexus-private-ip>:8085
        DOMAIN         = 'beepaa.online'
        // ANSIBLE_IP and NEXUS_IP are set as global environment variables in Jenkins
        // (Manage Jenkins > System > Global properties). Values come from `terraform output`.
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn -B clean compile org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=petclinic'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        // Slow on the first run (downloads the vulnerability database). Re-enable after the first green run.
        // stage('Dependency Check') {
        //     steps {
        //         dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //     }
        // }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps .'
            }
        }
        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-creds',
                groupId: 'Petclinic',
                nexusUrl: "${env.NEXUS_IP}:8081",
                nexusVersion: 'nexus3',
                protocol: 'http',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }
        stage('Trivy fs Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/petclinicapps'
            }
        }
        stage('Trivy image Scan') {
            steps {
                sh 'trivy image $NEXUS_REPO/petclinicapps > trivyimage.txt'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no ansible/deploy.yml ec2-user@$ANSIBLE_IP:/etc/ansible/deploy.yml
                        ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_IP \
                          "ansible-playbook -i /etc/ansible/stage-hosts /etc/ansible/deploy.yml -e nexus_repo=$NEXUS_REPO -e nexus_user=$NEXUS_USER -e nexus_password=$NEXUS_PASSWORD"
                    '''
                }
            }
        }
        stage('Check stage website availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' https://stage.${DOMAIN}", returnStdout: true).trim()
                    echo "Stage returned HTTP ${response}"
                    if (response != '200') {
                        error "Stage petclinic is not healthy (HTTP ${response})"
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@$ANSIBLE_IP \
                          "ansible-playbook -i /etc/ansible/prod-hosts /etc/ansible/deploy.yml -e nexus_repo=$NEXUS_REPO -e nexus_user=$NEXUS_USER -e nexus_password=$NEXUS_PASSWORD"
                    '''
                }
            }
        }
        stage('Check prod website availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' https://prod.${DOMAIN}", returnStdout: true).trim()
                    echo "Prod returned HTTP ${response}"
                    if (response != '200') {
                        error "Prod petclinic is not healthy (HTTP ${response})"
                    }
                }
            }
        }
    }
}
