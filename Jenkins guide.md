### Get the Output of a Shell Command in a Jenkins Pipeline


3. Using the sh returnstdout
The sh step command is a built-in Jenkins pipeline step that allows developers to execute a shell command and capture its output into a variable. Using shell commands, we can assign variables to strings. In a Jenkins pipeline, we can use the sh step to execute a shell command within a pipeline. Let’s look at a pipeline job to execute a command and store its output:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def output = sh(returnStdout: true, script: 'pwd')
                    echo "Output: ${output}"
                }
            }
        }
    }
}
```
The pipeline job above defines a stage “Example” along with its steps to execute, including the script element. This special step enables the execution of arbitrary Groovy code within the pipeline. Also, it is utilized to define a Groovy closure that captures shell output. Here, we performed the pwd command inside the script element of the pipeline using the sh returnStdout method. Also, we stored the result in the output variable. The option “returnStdout: true” instructs Jenkins to return the standard output of the shell command. The option “script: ‘pwd'” specifies the shell command to execute.
Finally, the echo command prints the value of the “output” variable to the Jenkins console using the echo step.
4. Using Command Substitution
Command substitution is a shell feature that allows developers to execute a command within a string and use its output as part of the string. In the Jenkins pipeline, we can use command substitution to capture the output of a shell command into a variable:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def output = sh(script: "echo \$(ls)", returnStdout: true)
                    echo "Output: ${output}"
                }
            }
        }
    }
}
```
In this case, the sh step executes the command “echo $(ls)” and captures its output into a variable using command substitution. Furthermore, the pipeline job captures the output of the “ls” command into the “output” variable and prints it to the Jenkins console upon execution.
5. Other Possible Cases
Jenkins pipeline provides two other important options which are very useful in storing the result to further use in the pipeline job. The returnStatus method captures the exit status of a shell command, whereas returnStdoutTrim is helpful in trimming the output for further use.
5.1. Using the returnStatus
The Jenkins pipeline’s sh step uses the returnStatus option to capture the exit status of a shell command. By default, the sh step returns the standard output of the shell command as the result of the step. However, when returnStatus is set to true, the exit status of the shell command is returned instead of the standard output.
Let’s look at the pipeline job with returnStatus method:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                    def status = sh(returnStatus: true, script: 'ls /test')
                    if (status != 0) {
                        echo "Error: Command exited with status ${status}"
                    } else {
                        echo "Command executed successfully"
                    }
                }
            }
        }
    }
}
```
In this example, the sh step executes the ls command and captures the output into the status variable using returnStatus. Since the directory /test doesn’t exist, the ls command will exit with a non-zero status, indicating an error. The if statement checks if the status variable is non-zero, and if so, prints an error message to the Jenkins console.
Developers can perform error checking by capturing the exit status of shell commands. Furthermore, they can also implement logic based on the success or failure of a command.
5.2. Using the Trim Method
In the following snippet The returnStdout parameter is set to true, which means that the standard output of the command will be captured and returned as the result of the step. Additionally, we can also use the trim() method to get the cleansed output of the command in a variable:
```
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                script {
                  def output = sh(returnStdout: true, script: 'echo "    hello    "').trim()
                  echo "Output: '${output}'"
                }
            }
        }
    }
}
```
The trim() method will remove any leading and trailing whitespace characters from the output of the echo command. Furthermore, the Jenkins console will display the resulting output enclosed in single quotes to highlight any whitespace characters.

---
### Jenkins sample
##### **Sample 1**
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


##### **Sample 2**
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


##### **Code to get hash commit**
```
pipeline {
    agent any
    stages {
        stage("Get hash") {
            steps {
                dir("/var/jenkins_home/workspace/My first pipeline/aws-tinhocthatladongian-webserver") {
                    script {
                        CURRENT_HASH = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    }
                    echo "${CURRENT_HASH}"
                }
            }
        }
    }
}
```
##### **Other way (define in function)**
```
pipeline {
    agent any
    stages {
        stage("Get hash") {
            steps {
                dir("/var/jenkins_home/workspace/My first pipeline/aws-tinhocthatladongian-webserver") {
                    script{
                        def CURRENT_HASH = getHash()
                    }
                    echo "${CURRENT_HASH}"
                }
            }
        }
    }
}

def getHash(){
    script{
        CURRENT_HASH = sh(script:'git rev-parse HEAD', returnStdout: true)
    }
    return CURRENT_HASH
}
```
