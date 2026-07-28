pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "rajeevgangaraju/simple-app"
        SONAR_ORG    = "rajeevgangaraju"
        SONAR_PROJ   = "simple-app"
    }

    stages {

        stage('Environment Check') {
            steps {
                sh '''
                    java -version
                    javac -version
                    mvn -version
                '''
            }
        }

        stage('Maven Compile & Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarCloud Scan Validation') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    withCredentials([
                        string(
                            credentialsId: 'sonarcloud-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh """
                        mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.token=\$SONAR_TOKEN \
                        -Dsonar.organization=${SONAR_ORG} \
                        -Dsonar.projectKey=${SONAR_PROJ}
                        """
                    }
                }
            }
        }

        stage('Verify Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()

                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }

                        echo "Quality Gate Passed"
                    }
                }
            }
        }

        stage('Execute Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                """
            }
        }

        stage('Push Docker Image') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh """
                    echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin

                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {

                withCredentials([
                    string(
                        credentialsId: 'git-pat',
                        variable: 'GIT_TOKEN'
                    )
                ]) {

                    sh """
                    rm -rf manifests

                    git clone https://\$GIT_TOKEN@github.com/rajeevgangaraju/app-manifests-repo.git manifests

                    cd manifests

                    git config user.email "jenkins-bot@poc.com"
                    git config user.name "Jenkins"

                    sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|' deployment.yaml

                    git add deployment.yaml

                    git commit -m "Updated image tag to ${BUILD_NUMBER}" || true

                    git push origin main
                    """
                }
            }
        }
    }

    post {

        success {
            echo "Pipeline Completed Successfully"
        }

        failure {
            echo "Pipeline Failed"
        }
    }
}
