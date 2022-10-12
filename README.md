# Java-spring-tomcat-application CI-CD

<p align="center">
<img width="319" alt="speing" src="https://user-images.githubusercontent.com/65728956/195344100-58b87a58-83cd-4e83-b1e4-20c74f1f6f44.png">
</p>

In this project we are going a CI-CD pipeline on ```Jenkins``` to deploy ```Java Spring``` Application on a ```Tomcat``` container using ```Kubernetes```.

## Tools used to build pipeline

- [Jenkins](https://www.jenkins.io/) - To build CI-CD pipeline
- [Gradle](https://gradle.org/) - To build the Java package 
- [SonarQube](https://www.sonarqube.org/) - To do static Code Analysis
- [Nexus](https://www.sonatype.com/) - To store the Artifacts
- [Docker](https://www.docker.com/) - To build and push docker image 
- [Datree](https://www.datree.io/) - To validate the helm charts
- [Kubernetes](https://kubernetes.io/) - To deploy the Java application on Tomcat container

## Pipeline stages

### 1) Sonar Quality Check for static code analysis
```sh
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
```
### 2) Build and Push Docker image to Nexus Artifactory
```sh
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
```
### 3) Validate Helm chart configuration
```sh
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
```
### 4) Stage to take manual approval before deployment
```sh
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
```
### 5) Deployment on Kubernetes
```sh
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
```
### 6) Verify if the deployment is successful 
```sh
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
```
### 7) Post block to sent the notification to Slack Channel 
```sh
post{
    success{
        slackSend channel: 'yash-jenkins', message: "Jenkins job ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER} has SUCCEDED..! Build Url: ${env.BUILD_URL}, Build Result: ${currentBuild.result}" , teamDomain: 'yash-ybc8444', tokenCredentialId: 'slack-token'
    }
    failure{
        slackSend channel: 'yash-jenkins', message: "Jenkins job ${env.JOB_NAME}, Build Number: ${env.BUILD_NUMBER} has FAILED..!  Build Url: ${env.BUILD_URL}, Build Result: ${currentBuild.result}" , teamDomain: 'yash-ybc8444', tokenCredentialId: 'slack-token'
    }
}
```



