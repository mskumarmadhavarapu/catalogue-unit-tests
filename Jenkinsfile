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
        stage('Dependabot Alerts Check') {
            steps {
                script {
                    def owner  = 'mskumarmadhavarapu'
                    def repo   = 'catalogue-unit-tests'
                    def apiUrl = "https://api.github.com/repos/${owner}/${repo}/dependabot/alerts"

                    withCredentials([usernamePassword(
                        credentialsId: 'github-token',  // ← your Jenkins credential ID
                        usernameVariable: 'GH_USER',
                        passwordVariable: 'GH_TOKEN'
                    )]) {
                        def foundAlerts = []

                        ['high', 'critical'].each { severity ->
                            def response = sh(
                                script: """
                                    curl -sf -w '\nHTTP_STATUS:%{http_code}' \
                                    -H 'Authorization: Bearer ${GH_TOKEN}' \
                                    -H 'Accept: application/vnd.github+json' \
                                    -H 'X-GitHub-Api-Version: 2022-11-28' \
                                    '${apiUrl}?severity=${severity}&state=open&per_page=100'
                                """,
                                returnStdout: true
                            ).trim()

                            def parts  = response.split('\nHTTP_STATUS:')
                            def status = parts[1].toInteger()
                            def body   = parts[0]

                            if (status == 403) {
                                error("GitHub API 403 — verify token has 'security_events' scope " +
                                    "and Dependabot alerts are enabled on the repo.")
                            }
                            if (status != 200) {
                                error("GitHub API returned HTTP ${status}")
                            }

                            def alerts = readJSON(text: body)

                            alerts.each { a ->
                                def pkg  = a.dependency?.package?.name ?: 'unknown'
                                def ghsa = a.security_advisory?.ghsa_id ?: 'n/a'
                                def fix  = a.security_vulnerability
                                            ?.first_patched_version?.identifier ?: 'no fix'
                                foundAlerts << "[${severity.toUpperCase()}] #${a.number} | " +
                                            "${pkg} | ${ghsa} | fix: ${fix}"
                            }

                            echo alerts.size() > 0
                                ? "⚠️  ${alerts.size()} ${severity.toUpperCase()} alert(s) found"
                                : "✅ No ${severity.toUpperCase()} alerts"
                        }

                        if (foundAlerts.size() > 0) {
                            echo "\n❌ Dependabot check FAILED — ${foundAlerts.size()} alert(s):\n" +
                                foundAlerts.join('\n')
                            error("Pipeline blocked: ${foundAlerts.size()} HIGH/CRITICAL " +
                                "Dependabot alert(s) detected. Resolve before merging.")
                        }

                        echo '✅ Dependabot check passed — no HIGH or CRITICAL alerts.'
                    }
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