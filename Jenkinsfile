def getAWSUser() {
    return env.BUILD_USER_ID
}

def getCredentialsId() {
    return env.BUILD_USER_ID
}

def assumeRole(String credentials: "aws", String userName,
  String accountId = "YOUR_AWS_ACCOUNT_ID", String role = "role_to_be_assumed") {
  def String trustedAccount = "YOUR_AWS_ACCOUNT_ID"

  def mfa = input(
    message: "Enter MFA Token",
    parameters: [[$class: 'StringParameterDefinition', name: 'mfa', trim: true]]
  )

  withCredentials([[
    $class: 'AmazonWebServicesCredentialsBinding',
    credentialsId: '${credentials}',
    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
  ]]) {
    return withAWS(role: 'ecr', roleAccount: '${ACCOUNT_ID}', externalId: 'externalId') {
		withAWS(role: 'ecr', roleAccount: '${ACCOUNT_ID}', externalId: 'externalId') {
                                sh"""
                                    aws sts get-caller-dentity
                                """
                        }

           }
  }
}


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
	stage('credentials') {
     	    steps {
                script {
                    jsonCreds = assumeRole("aws", "session-role")
                    creds = readJSON text: "${jsonCreds}"
         	}       
		
            }
        }

	stage('Use credentials') {
            steps {
        	withEnv([
            	    "AWS_ACCESS_KEY_ID=${creds.AccessKeyId}",
            	    "AWS_SECRET_ACCESS_KEY=${creds.SecretAccessKey}",
            	    "AWS_SESSION_TOKEN=${creds.SessionToken}"
          	]) {
		        withAWS(role: 'ecr', roleAccount: '${ACCOUNT_ID}', externalId: 'externalId') {
                    		sh"""
                    		    aws sts get-caller-dentity
				"""
                	}

        	}
	    }

    	}	

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
