pipeline {
    agent any

    environment {
        registry = "imranvisualpath/vproappdock"
        registryCredential = "dockerhub"
    }

    stages {
        stage("BUILD") {
            steps {
                sh "mvn clean install -DskipTests"
            }
            post {
                success {
                    echo "Now Archiving..."
                    archiveArtifacts artifacts: "**/target/*.war"
                }
            }
        }

        stage("UNIT TEST") {
            steps {
                sh "mvn test"
            }
        }

        stage("INTEGRATION TEST") {
            steps {
                sh "mvn verify -DskipUnitTests"
            }
        }

        stage("CODE ANALYSIS WITH CHECKSTYLE") {
            steps {
                sh "mvn checkstyle:checkstyle"
            }
            post {
                success {
                    echo "Generated Analysis Result"
                }
            }
        }

        stage("BUILD IMAGE") {
            steps {
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }

        stage("DEPLOY IMAGE") {
            steps {
                script {
                    docker.withRegistry("", registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage("REMOVE UNUSED DOCKER IMAGE") {
            steps {
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        }

        stage("CODE ANALYSIS W/ SONARQUBE") {
            environment {
                scannerHome = tool "mysonarscanner4"
            }

            steps {
                withSonarQubeEnv("sonar-pro") {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("KUBERNETES DEPLOY") {
            agent {label "KOPS"}
            
            steps {
                sh "helm upgrade --install --force vprofile-satck helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}