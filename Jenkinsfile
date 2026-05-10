pipeline {
    agent {
        node {
            label 'ROBOSHOP'
        }
    }
    environment { 
        appVersion = ""
        ACC_ID = "520072122793"
        region = "us-east-1"
    }
    options { 
        disableConcurrentBuilds()
        // timeout(time: 5, unit: 'MINTUES')
    }
    // parameters {
    //     string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    //     text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
    //     booleanParam(name: 'DEPLOY', defaultValue: false, description: 'Toggle this value')
    //     choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
    //     password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    // }
    stages {
        stage('Read version') {
            steps {
                script {
                    // Reads the file into a Groovy Map
                    def packageJson = readJSON file: 'package.json'
                    
                    // Access specific fields
                    appVersion = packageJson.version
                    echo "Building version ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                    sh """
                        npm install 
                    """
                }
            }
        }
        stage('Unit tests') {
            steps {
                script{
                    sh """
                        npm test
                    """
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool name: 'sonar-8' // agent configuration
                    withSonarQubeEnv('sonar-server') { // analysing and uploading to server
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
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
        stage('Build Image') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${region}") {
                        // Commands here use the authorized environment
                        sh """
                            aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """
                    }
                }
            }
        }
    }
    // post build
    post { 
        always { 
            echo 'I will always say Hello again!'
        }
        success { 
            echo 'pipeline success'

        }
        failure { 
            echo 'pipeline failure'
        }
    }
}