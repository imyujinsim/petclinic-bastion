pipeline {
    agent any

    tools {
      // Jenkins 'Global Tool Configuration' 에 설정한 버전과 연동
      maven "M3"
    }

    environment {
        ECR_PATH = '851557167064.dkr.ecr.ap-northeast-2.amazonaws.com'
        ECR_IMAGE = 'demo-maven-springboot'
        REGION = 'ap-northeast-2'
        ACCOUNT_ID='851557167064'
    }

    stages {
        stage('Git Clone from gitSCM') {
            steps {
                script {
                    try {
                        git branch: 'main', 
                            credentialsId: 'GitCredential',
                            url: 'https://github.com/imyujinsim/petclinic-bastion'
                        sh "ls -lat"
                        env.cloneResult=true
                        
                    } catch (error) {
                        print(error)
                        env.cloneResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage("Build JAR with Maven") {
            when {
                expression {
                    return env.cloneResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }                
            }
            steps {
                script{
                    try {
                        sh """
                        rm -rf deploy
                        mkdir deploy
                        java -version
                        """
			sh 'export AWS_ACCESS_KEY_ID=ASIA4MRGDF7MJQPBNZFB'
			sh "export AWS_SECRET_ACCESS_KEY=qe4XppwFSdy0pl18SP5bRAfHRLCqeeWCQ7WXEr2n"
			sh "export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEDwaDmFwLW5vcnRoZWFzdC0yIkcwRQIhALfK4XSXsnHjgBfOk/iPpyAlb3zRY1HpKnlLKz72el0CAiB1J9ehEjSpln+rrIh4AjAva2R0QI5wJJKbPAhhNtQsfyrvAQhFEAEaDDg1MTU1NzE2NzA2NCIM9Iaw0ce0XwFtCYnEKswBjYIXDTto7qU8ZGAIMTbrK1Uhx6GczL4JOaj8KgtvhqLizv71AzN4OfyjhQKpiRg5ocjTTp5GTCYFipyuRVdVqokREgEjrO4nOezN6pVVHU3uB4hGGUvINFhlK2kFVR7QOb3Mhd2sZcQpN0DIoE7BfuABUzqj6auKRwGaz6orx7zxXJLH2JLZcXAN1VRQQb4k3VzfD6/OkYz4be2TyEKvUt/SXD7LQtJwPZpYLYDmVDjOa9HDf9m68X4zmB5Ht77qGmVRvKMn0TnDFuN7MJbv3ZsGOpgBlAyMsSUKwiV6GcWg/UgEP/BYkLbv1Ny7r3cP5FtNBO+GUfjTnNxnVH866E/tfmZQ2jOuMYxONkUPUSjtXYfuzhdYApc54Z9Cf/5AReg3a3Hdhcxny6QCA8vtx9UfnzyTlCPC/kiyRm7BEkqK1aVRa8wH+8T/OJqZzz48hqwGVBlLUidFrKjyIzHDJSQ6n4EKn9srNpJQ9hk="
                        // sh "sed -i 's/  version:.*/  version: \${VERSION:v${env.BUILD_NUMBER}}/g' /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.yaml"
                        // sh "cat /var/lib/jenkins/workspace/${env.JOB_NAME}/src/main/resources/application.yaml"
                        sh './mvnw package'
                        sh """
                        cd deploy
                        cp /var/jenkins_home/workspace/${env.JOB_NAME}/target/*.jar ./${ECR_IMAGE}.jar
                        """
                        env.mavenBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Failed to build jar file'
                        env.mavenBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
        stage('Docker Build and Push to ECR'){
            when {
                expression {
                    return env.mavenBuildResult ==~ /(?i)(Y|YES|T|TRUE|ON|RUN)/
                }
            }
            steps {
                script{
                    try {
                        sh"""
                        #!/bin/bash
                        cat>Dockerfile<<-EOF
FROM openjdk:11-jre-slim
ENV JAVA_OPTS="-XX:InitialRAMPercentage=40.0 -XX:MaxRAMPercentage=80.0"
ADD ./${ECR_IMAGE}.jar /home/${ECR_IMAGE}.jar
CMD nohup java -jar -Dspring.profiles.active="mysql" /home/${ECR_IMAGE}.jar 1> /dev/null 2>&1
EXPOSE 8080
EOF"""
                        docker.withRegistry("${ECR_PATH}/petclinic", "ecr:ap-northeast-2:aws") {
                            def image = docker.build("${ECR_PATH}/${ECR_IMAGE}:${env.BUILD_NUMBER}")
                            image.push()
                        }
                        
                        echo 'success-remove Deploy Files'
                        // sh "sudo rm -rf /var/jenkins_home/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=true
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "sudo rm -rf /var/jenkins_home/workspace/${env.JOB_NAME}/*"
                        env.dockerBuildResult=false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }
    }
}
