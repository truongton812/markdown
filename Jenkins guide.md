
---
**Sample 1**
```
#!/usr/bin/env groovy
pipeline {
    agent any #agent tag, which specify which agent would run this pipeline
    environment {
        demo = "SECRET"
    }
    stages {
        stage('build') {
             agent {
   Label ‘aws-amazon’ #specify agent with label ‘aws-amazon’ would run this stage
}
            when {
                expression { BRANCH_NAME ==~ /(test|staging)/ }
            }
            steps {
                docker_loggin("docker_repository","docker.medpro.com.vn")
                docker_helper("docker.medpro.com.vn","thanhtruc" , "client")
                docker_helper("docker.medpro.com.vn","thanhtruc" , "server")
            }
        }
//        stage('test'){
//            steps{
//                test()
//           }
//        }
        stage('deploy'){
            when {
                expression { BRANCH_NAME ==~ /(test|staging)/ }
            }
            steps {
                docker_deploy()
            }
        }
    }
}

//helper

/// Docker help to deploy the image to docker hub .
def docker_helper(repository ,name , folder ) {
    sh """docker build -t $repository/$name -f $folder/Dockerfile ./$folder"""
}
def docker_loggin(credentials, url ) {
    withCredentials([usernamePassword(credentialsId: """$credentials""" ,usernameVariable: """repository_username""", passwordVariable: """repository_password""")]){
        sh """docker login $url -u $repository_username -p $repository_password """
    }
### take the credential from jenkins based on credential id, then extract username and password by built-in variable usernameVariable and passwordVariable and bind to 2 variables repository_username and repository_password
}
def docker_deploy() {
    sh 'docker-compose down '
    sh 'docker-compose up -d --no-color --wait --build --force-recreate'
    sh 'docker-compose ps '
}

def test() {
    def result = sh(script:"""curl -I http://localhost:8000 2>/dev/null | head -n 1 | awk '{print "${2}"}'""",returnStdout: true).trim()
    echo '$result'
    if (result != 200 ) {
        currentBuild.result = 'FAILURE'
    } else {
        currentBuild.result = 'SUCCESS'
    }
}
```

---
**Sample 2**
```
#!/usr/bin/env groovy
pipeline {
    agent {
        label "linux1"
    }
    // environment {
    //     TERRAFORM = tool name: 'terraform', type:  'com.cloudbees.jenkins.plugins.customtools.CustomTool'
    // }
    tools {
          terraform "terraform"
          gradle "gradle"
    }

    parameters {
        choice(
      name: 'ENV',
      choices: [" " ,'DEV', 'PROD'],
      description: 'Passing the Environment'
    )
    }
    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {

        stage('build') {
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script {
                def tagImage = getTagVersion()
                docker_loggin("docker_repository","docker.medpro.com.vn")
                docker_helper("docker.medpro.com.vn","client" , "client",tagImage)
                docker_helper("docker.medpro.com.vn","server" , "server",tagImage)
                }
            }
        }
        stage("Lint and Build helm Chart with Tag "){
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script {
                def tagHelmChart = getTagVersion()
                sh "gradle helmPackage -PChartVersion=$tagHelmChart"
                withCredentials([usernamePassword(credentialsId: 'gradleconfig' ,usernameVariable: 'username', passwordVariable: 'password')]){
                    sh "gradle  helmPublish -Pusername=$username -Ppassword=$password  -PChartVersion=$tagHelmChart"
                }
            }
        }
        }
        stage('deploy'){
            when {
                allOf {
                    buildingTag()
                }
            }
            steps {
                script{
                    dir("k8s"){
                        k8s_deploy()
                    }
                }
            }

        }
    }
}

//helper

/// Docker help to deploy the image to docker hub .
def docker_helper(repository ,name , folder ,tagImage) {
    sh "docker build -t $repository/$name:$tagImage -f $folder/Dockerfile ./$folder"
    sh "docker push $repository/$name:$tagImage"
}
def docker_loggin(credentials, url ) {
    withCredentials([usernamePassword(credentialsId: """$credentials""" ,usernameVariable: """repository_username""", passwordVariable: """repository_password""")]){
        sh """docker login $url -u $repository_username -p $repository_password """
    }
}
// if branch = dev config = devconfig , else kubeproduction

def k8s_deploy() {
    withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
        sh 'mkdir -p ~/.kube'
        sh 'cp $KUBECONFIG ~/.kube/config'
        sh "kubectl apply -f ."
}
}
def test() {
    def result = sh(script:"""curl -I http://localhost:8000 2>/dev/null | head -n 1 | awk '{print "${2}"}'""",returnStdout: true).trim()
    echo '$result'
    if (result != 200 ) {
        currentBuild.result = 'FAILURE'
    } else {
        currentBuild.result = 'SUCCESS'
    }
}
def getTagVersion(){
    def tagImage = params.ENV == ''  ? "${env.GIT_COMMIT}" : sh(returnStdout:  true, script: "git tag --contains").trim()
    return tagImage
}
```
