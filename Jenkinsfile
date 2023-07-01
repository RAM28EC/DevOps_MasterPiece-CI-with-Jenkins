pipeline {
    agent any

    environment {
        NAME = "spring-app"
        VERSION = "${env.BUILD_ID}"
        GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
        IMAGE_REPO = "jenkinsargocd"
        GIT_REPO_NAME = "DevOps_MasterPiece-CD-with-argocd"
        GIT_USER_NAME = "RAM28EC"
       
    }

    tools { 
        maven 'Maven3' 
    }
    stages {
        stage('Checkout git') {
            steps {
              git branch: 'main', url:'https://github.com/RAM28EC/DevOps_MasterPiece-CI-with-Jenkins.git'
            }
        }
        
        stage ('Build & JUnit Test') {
            steps {
                sh 'mvn clean install' 
            }
        }

        stage('SonarQube Analysis'){
            steps{
                withSonarQubeEnv('sonar-env') {
                        sh '''mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=devops \
                        -Dsonar.projectName='oraganization' \
                        -Dsonar.host.url=http://43.205.215.172:9000 \
                        -Dsonar.login=sqp_9ea13e5d71cdf019afc843785f3dd2af69adfe6e'''
                }
            }
        }

        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
        
        stage('Deploy to Artifactory') {
            environment {
                // Define the target repository in Artifactory
                TARGET_REPO = 'my-local-repo'
            }
            
            steps {
                script {
                    try {
                        def server = Artifactory.newServer url: 'http://13.126.17.214:8082/artifactory', credentialsId: 'jfrog'
                        def uploadSpec = """{
                            "files": [
                                {
                                    "pattern": "target/*.jar",
                                    "target": "${TARGET_REPO}/"
                                }
                            ]
                        }"""
                        
                        server.upload(uploadSpec)
                    } catch (Exception e) {
                        error("Failed to deploy artifacts to Artifactory: ${e.message}")
                    }
                }
            }
        }
        
        stage('Docker  Build') {
            steps {
               
      	         sh 'docker build -t ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT} .'
                
            }
        }

        stage('Docker Image Scan') {
            steps {
      	        sh ' trivy image --format template --template "@/usr/local/share/trivy/templates/html.tpl" -o report.html ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT} '
            }
        }    
        
        stage('Upload Scan report to AWS S3') {
              steps {
                  
                //  sh 'aws configure set aws_access_key_id "$AWS_ACCESS_KEY_ID"  && aws configure set aws_secret_access_key "$AWS_ACCESS_KEY_SECRET"  && aws configure set region ap-south-1  && aws configure set output "json"' 
                  sh 'aws s3 cp report.html s3://storeimagereport/'
              }
        }
        
        stage ('Docker Image Push') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'docker', gitToolName: 'Default')]) {
                     sh "docker login -u ${username} -p ${password} "
                     sh 'docker push ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                     sh 'docker rmi  ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                } 
            }
        }
        
        stage('Clone/Pull k8s deployment Repo') {
            steps {
                script {
                    if (fileExists('DevOps_MasterPiece-CD-with-argocd')) {

                        echo 'Cloned repo already exists - Pulling latest changes'

                        dir("DevOps_MasterPiece-CD-with-argocd") {
                          sh 'git pull'
                        }

                    } else {
                        echo 'Repo does not exists - Cloning the repo'
                        sh 'git clone -b feature https://github.com/RAM28EC/DevOps_MasterPiece-CD-with-argocd.git'
                    }
                }
            }
        }
        
        stage('Update deployment Manifest') {
            steps {
                dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                    sh 'eksctl create -f deployment.yaml'
                    sh 'cat deployment.yaml'
                }
            }
        }
        
        stage('Commit & Push changes to feature branch') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                        sh "git config --global user.email 'ram.28ec@gmail.com'"
                        sh 'git remote set-url origin https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}'
                        sh 'git checkout feature'
                        sh 'git add deployment.yaml'
                        sh "git commit -am 'Updated image version for Build- ${VERSION}-${GIT_COMMIT}'"
                        sh 'git push origin feature'
                    }
                }    
            }
        }
        
        stage('Raise PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("DevOps_MasterPiece-CD-with-argocd/yamls") {
                        sh '''
                            set +u
                            unset GITHUB_TOKEN
                            gh auth login --with-token < token.txt
                            
                        '''
                        sh 'git branch'
                        sh 'git checkout feature'
                        sh "gh pr create -t 'image tag updated' -b 'check and merge it'"
                    }
                }    
            }
        }    

    }
}


