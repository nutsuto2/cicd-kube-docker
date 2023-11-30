pipeline {
    agent any

    environment {
        registry = "2umm3r/vprofileapp"
        registryCredential = "dockerhub"
    }

    stages{
        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('Code Analysis with Checkstyle'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('Code Analysis with SonarQube'){
            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps{
                withSonarQubeEnv('spnar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image'){
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_ID"
                }
            }
        }

        stage('Upload Image'){
            steps{
                script {
                    docker.withRegistry('', registryCredential){
                        dockerImage.push("V$BUILD_ID")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Remove Unsued Image'){
            steps{
                sh "docker rmi $registry:V$BUILD_ID"
            }
        }

        stage('Kubernetes Deploy'){
            agent {label 'KOPS'}
            steps {
                sh 'helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUID_ID} --namespace prod'
            }
        }
    }
}