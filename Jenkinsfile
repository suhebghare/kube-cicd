pipeline{
   agent any

    environment {
        registry = "suhebghare/vpapp"
        registryCredential = "kube-docker-login"
    }

   stages{

       stage ('Build'){
            steps{
                sh 'mvn clean install -DskipTests'
            }
            post{
                success{
                    echo 'Archiving now...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
       }

       stage('UNIT TEST'){
           steps {
               sh 'mvn test'
           }
       }

       stage('INTEGRATION TEST'){
           steps {
               sh 'mvn verify -DskipUnitTests'
           }
       }

        stage('code analysis with checkstyle'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post{
                success{
                    echo 'analysis result generated'
                }
            }
        }

        stage ('code analysis with sonarqube'){

            environment {
                scannerHome = tool 'sonar-4.4.0'
            }

            steps{
                withSonarQubeEnv('sonarqube') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit:'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build app image'){
            steps{
                script{
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('Upload Image'){
            steps{
                script{
                    docker.withRegistry('',registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Remove Old Image'){
            steps{
                sh 'docker rmi $registry:V$BUILD_NUMBER'
            }
        }

        stage('K8s Deploy'){
            agent {label 'KOPS'}

            steps{
                sh "helm upgrade --install --force --namespace prod vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER}"
            }
        }
   }
}
