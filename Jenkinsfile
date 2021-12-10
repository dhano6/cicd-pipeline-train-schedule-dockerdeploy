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
                // script znamena ze nepouzijem declarative syntax lebo toto sa neda urobit cez declarat ale scripted
                script {
                    // build image
                    app = docker.build("dhano6/train-schedule")
                    // smoke test image - run image and run following command inside image 
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
                    // docker_hub_login is already created credential kind: username/password
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        // pushing tags
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
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
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        // sshpass je CLI program nainstalovany na servri tzn. nie jenkins plugin
                        // sluzi nato ze spustis ssh command a zadas heslo cez argument teda nie interaktivne
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull dhano6/train-schedule:${env.BUILD_NUMBER}\""
                        // try block je nato ze tie komandy v nom mozu niekedy failnut ale to nieje na zavadu a ja chcem pokracovat, teda nechcem aby spadla piplina
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d dhano6/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
