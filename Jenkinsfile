pipeline {
    agent any
    environment {
        registryCredential = "ecr:us-east-1:awscreds"
        appRegistry = "767397682018.dkr.ecr.us-east-1.amazonaws.com/vprofileapprepo"
        vprofileRegistry = "https://767397682018.dkr.ecr.us-east-1.amazonaws.com"
        cluster = "vprofilecluster"
        service = "vprofileappsvc"
    }

    stages {
        stage ("Code Fetch") {
            steps {
                git branch : "docker", url : "https://github.com/devopshydclub/vprofile-project.git"
            }
        }

        stage ("Unit Test") {
            steps {
                sh "mvn test"
            }
        }

        stage ("Checkstyle Analysis") {
            steps {
                sh "mvn checkstyle:checkstyle"
            }
            post {
                success {
                    echo "Generated Analysis Result"
                }
            }
        }

        stage ("SonarQube Analysis") {
            environment {
                scannerHome = tool 'sonar4.7'
            }
            steps {
                withSonarQubeEnv ('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xm'''
                }
            }
        }

        stage ("Quality Gate") {
            steps {
                timeout(time: 10 , unit: 'MINUTES') {
                    waitForQualityGate abortPipeline : true
                }
            }
        }

        stage ("Build Docker App Image") {
            steps {
                script {
                    dockerImage = docker.build(appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                }
            }
        }

        stage ("Upload App Image to ECR") {
            steps {
                script {
                    docker.withRegistry (vprofileRegistry, registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage ("Deploy to ECS") {
            steps {
                withAWS (credentials: "awscreds", region: "us-east-1") {
                    sh "aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment"
                }
            }
        }
    }
}