pipeline {
    agent any
    stages{
        stage('Build'){
            steps{
                echo 'Runnning build automation'
		sh './gradlew build --no-daemon'
		archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('DeployStaging'){
			when {
				branch 'master'
        	}
			steps{
				withCredentials([usernamePassword(credentialsId: 'deploy_user',usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
					sshPublisher(
						failOnError: true,
						continueOnError: false,
						publishers: [
							sshPublisherDesc(
								configName: 'staging',
								sshCredentials: [
									username: "$USERNAME",
									encryptedPassphrase: "$USERPASS"
									],
								transfers: [
									sshTransfer(
										sourceFiles: 'dist/trainSchedule.zip',	
										removePrefix: 'dist/',
										remoteDirectory: '/tmp',
										execCommand: 'sudo /usr/bin/systemctl stop train-schedule && sudo rm -rf /opt/train-schedule/* && sudo unzip /tmp/trainSchedule.zip -d /opt/train-schedule && sudo /usr/bin/systemctl start train-schedule'
										)
									]
								)
							]
						)
					}
				}
			}



        stage('DeployProduction'){
			when {
				branch 'master'
        	}
			steps{
				input 'Does staging look okay?'
				milestone(1)
				withCredentials([usernamePassword(credentialsId: 'deploy_user',usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
					sshPublisher(
						failOnError: true,
						continueOnError: false,
						publishers: [
							sshPublisherDesc(
								configName: 'production',
								sshCredentials: [
									username: "$USERNAME",
									encryptedPassphrase: "$USERPASS"
									],
								transfers: [
									sshTransfer(
										sourceFiles: 'dist/trainSchedule.zip',	
										removePrefix: 'dist/',
										remoteDirectory: '/tmp',
										execCommand: 'sudo /usr/bin/systemctl stop train-schedule && sudo rm -rf /opt/train-schedule/* && sudo unzip /tmp/trainSchedule.zip -d /opt/train-schedule && sudo /usr/bin/systemctl start train-schedule'
										)
									]
								)
							]
						)
					}
				}
			}

        }
    }



pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("jesterboxboy82/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToStaging') {
            when {
                branch 'master'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'deploy_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$staging_ip \"docker pull jesterboxboy82/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$staging_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$staging_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$staging_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d jesterboxboy82/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'deploy_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull jesterboxboy82/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d jesterboxboy82/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
