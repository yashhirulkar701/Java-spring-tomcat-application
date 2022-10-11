pipeline{
    agent any

    tools {
        gradle 'gradle'
    }

    environment {
        VERSION = "${BUILD_ID}"
        IMAGE_NAME = "spring-app"
    }

    stages{
        stage("Sonar Quality Check"){
            steps{
                script{
                    withSonarQubeEnv('sonar-server') {
                        sh '''
                        chmod +x gradlew
                        ./gradlew sonarqube
                        '''
                    }
                    timeout(30) {
                        def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                    }
                }  
            }
        }
        stage("Docker build & Push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus-pass', variable: 'nexus_password')]) {
                        sh '''
                            docker build -t 13.232.228.68:8082/${IMAGE_NAME}:${VERSION} .
                            docker login -u admin -p $nexus_password 13.232.228.68:8082
                            docker push 13.232.228.68:8082/${IMAGE_NAME}:${VERSION}
                            docker image rm  13.232.228.68:8082/${IMAGE_NAME}:${VERSION}
                        '''
                    }
                }  
            }
        }
        stage("Helm charts validation"){
            steps{
                script{
                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=7d7e3c05-722a-4da1-b092-dcd69f6a31ff']) {
                            sh '''
                            helm plugin install https://github.com/datreeio/helm-datree || true
                            helm datree test myapp/
                            '''
                        }
                    }
                }
            } 
        }
        stage("Push helm charts to Nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus-pass', variable: 'nexus_password')]) {
                        dir('kubernetes/') {
                            sh '''
                                helm_version=$(helm show chart myapp | grep version | cut -d ":" -f2 | tr -d " ")
                                tar -czvf myapp-${helm_version}.tgz myapp/
                                curl -u admin:$nexus_password http://13.232.228.68:8081/repository/helm-repo/ --upload-file myapp-${helm_version}.tgz -v
                            '''
                        }
                    }
                }
            }  
        }
        stage("Manual Approval"){
            steps{
                script{
                    timeout(5) {
                        slackSend channel: 'yash-jenkins', message: "Jenkins job ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER}. Please go to the build url and approve the deployment request..! Build Url: ${env.BUILD_URL}", teamDomain: 'yash-ybc8444', tokenCredentialId: 'slack-token'
                        input  id: 'Deploy Gate', message: 'Should we deploy..? ', ok: 'Yes, we should...!'
                    }
                }
            }  
        }
        stage("Deploying on Kubernetes"){
            steps{
                script{
                    withCredentials([file(credentialsId: 'KUBECONFIG', variable: 'KUBE_CONFIG')]) {
                        dir('kubernetes/') {
                            withCredentials([string(credentialsId: 'nexus-pass', variable: 'nexus_password')]) {
                                sh '''
                                export KUBECONFIG=$KUBE_CONFIG
                                kubectl create secret docker-registry registry-secret --docker-server=13.232.228.68:8082 --docker-username=admin --docker-password=$nexus_password --dry-run=client -o yaml | kubectl apply -f - 
                                docker pull 13.232.228.68:8082/${IMAGE_NAME}:${VERSION}
                                helm upgrade --install --set image.repository="13.232.228.68:8082/${IMAGE_NAME}" --set image.tag="${VERSION}"  myspringapp myapp/
                                '''
                            }
                        }
                    }
                }
            }  
        }
        stage('verifying app deployment'){
            steps{
                script{
                    withCredentials([file(credentialsId: 'KUBECONFIG', variable: 'KUBE_CONFIG')]) {
                        sh '''
                            export KUBECONFIG=$KUBE_CONFIG
                            kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myspringapp-myapp:8080
                        '''
                    }   
                }
            }
        }       
    }
    post{
        success{
            slackSend channel: 'yash-jenkins', message: "Jenkins job ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER} has SUCCEDED..! Build Url: ${env.BUILD_URL}, Build Result: ${currentBuild.result}" , teamDomain: 'yash-ybc8444', tokenCredentialId: 'slack-token'
        }
        failure{
            slackSend channel: 'yash-jenkins', message: "Jenkins job ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER} has FAILED..!  Build Url: ${env.BUILD_URL}, Build Result: ${currentBuild.result}" , teamDomain: 'yash-ybc8444', tokenCredentialId: 'slack-token'
        }
    }
}
