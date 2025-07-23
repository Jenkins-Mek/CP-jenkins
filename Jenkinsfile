properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        booleanParam(name: 'INCLUDE_INTERNAL', defaultValue: false, description: 'Include internal Kafka topics (starting with _)'),
        choice(name: 'OUTPUT_FORMAT', choices: ['list', 'json', 'csv'], description: 'Output format for topics list'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['PLAINTEXT', 'SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol')
    ])
])

pipeline {
    agent any

    environment {
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        TOPICS_JSON_FILE = 'kafka-topics.json'
        TOPICS_CSV_FILE = 'kafka-topics.csv'
        TOPICS_REPORT_FILE = 'kafka-topics-report.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Validate Environment') {
            steps {
                script {
                    echo "🔍 Validating Kafka environment..."
                    validateEnvironment()
                    echo "✅ Environment validation passed"
                }
            }
        }

        stage('Wait for Kafka Services') {
            steps {
                script {
                    echo "⏳ Waiting for Kafka services to be ready..."
                    waitForKafkaServices()
                    echo "✅ Kafka services are ready"
                }
            }
        }

        stage('Create Client Configuration') {
            steps {
                script {
                    echo "🔧 Creating Kafka client configuration..."
                    withCredentials([
                        usernamePassword(
                            credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9',
                            usernameVariable: 'KAFKA_USERNAME',
                            passwordVariable: 'KAFKA_PASSWORD'
                        )
                    ]) {
                        createKafkaClientConfig(params.COMPOSE_DIR, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                    echo "✅ Client configuration created securely"
                }
            }
        }

        stage('List Kafka Topics') {
            steps {
                script {
                    echo "📋 Retrieving all Kafka topics..."
                    def topics = listKafkaTopics()
                    
                    if (topics.size() > 0) {
                        echo "✅ Found ${topics.size()} topic(s)"
                        topics.eachWithIndex { topic, index ->
                            echo "  ${index + 1}. ${topic}"
                        }
                        
                        env.TOPICS_COUNT = topics.size().toString()
                        env.TOPICS_FOUND = topics.join(',')
                        saveTopicsAsArtifacts(topics)
                        
                    } else {
                        echo "⚠️ No topics found in the Kafka cluster"
                        handleEmptyTopics()
                    }
                }
            }
        }

        stage('Generate Summary Report') {
            steps {
                script {
                    echo "📊 Generating summary report..."
                    generateSummaryReport()
                    echo "✅ Summary report generated"
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ Successfully listed all Kafka topics (${env.TOPICS_COUNT} found)"
                archiveArtifacts artifacts: "${env.TOPICS_LIST_FILE},${env.TOPICS_JSON_FILE},${env.TOPICS_CSV_FILE},${env.TOPICS_REPORT_FILE}",
                               fingerprint: true,
                               allowEmptyArchive: true
                
                echo "📦 Topics artifacts archived and available for downstream pipelines"
                printUsageInstructions()
            }
        }
        failure {
            echo """❌ Failed to list topics. Troubleshooting steps:
1. Check if Kafka services are running: docker compose ps
2. Verify broker container is accessible
3. Check Kafka broker logs for errors
4. Validate bootstrap server configuration
5. Verify Kafka credentials in Jenkins secrets"""
        }
        always {
            script {
                cleanupClientConfig()
                echo "🏁 Kafka topics listing operation completed"
            }
        }
    }
}

// ========== HELPER FUNCTIONS ==========

def validateEnvironment() {
    if (!params.COMPOSE_DIR) {
        error("COMPOSE_DIR parameter is required")
    }
    
    def composeFile = "${params.COMPOSE_DIR}/docker-compose.yml"
    def checkResult = sh(
        script: "test -f ${composeFile} && echo 'exists' || echo 'missing'",
        returnStdout: true
    ).trim()
    
    if (checkResult == 'missing') {
        error("Docker compose file not found at: ${composeFile}")
    }
}

def waitForKafkaServices() {
    def maxRetries = 30
    def retryCount = 0
    def isReady = false
    
    while (retryCount < maxRetries && !isReady) {
        try {
            def result = sh(
                script: """
                    docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                    exec -T broker bash -c "echo 'Kafka broker is accessible'" 2>/dev/null
                """,
                returnStatus: true
            )
            
            if (result == 0) {
                isReady = true
            } else {
                retryCount++
                echo "⏳ Waiting for Kafka broker... (${retryCount}/${maxRetries})"
                sleep(10)
            }
        } catch (Exception e) {
            retryCount++
            echo "⏳ Kafka not ready yet... (${retryCount}/${maxRetries})"
            sleep(10)
        }
    }
    
    if (!isReady) {
        error("❌ Kafka services failed to start within timeout period")
    }
}

def createKafkaClientConfig(composeDir, username, password) {
    def securityConfig = ""
    
    switch(params.SECURITY_PROTOCOL) {
        case 'SASL_PLAINTEXT':
        case 'SASL_SSL':
            securityConfig = """
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            break
        case 'PLAINTEXT':
        default:
            securityConfig = """
security.protocol=PLAINTEXT
"""
            break
    }
    
    sh """
        echo "Creating secure client.properties file..."
        docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
        exec -T broker bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
${securityConfig}
EOF'
        
        echo "Verifying client configuration (secrets masked)..."
        docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
        exec -T broker bash -c "grep -v 'password' ${env.CLIENT_CONFIG_FILE} || echo 'Config created with secure credentials'"
    """
}

def listKafkaTopics() {
    def topicsOutput = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "
                export KAFKA_OPTS=''
                export JMX_PORT=''
                export KAFKA_JMX_OPTS=''
                unset JMX_PORT
                unset KAFKA_JMX_OPTS
                kafka-topics --list --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} --command-config ${env.CLIENT_CONFIG_FILE} 2>/dev/null
            "
        """,
        returnStdout: true
    ).trim()

    def allTopics = topicsOutput.split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') }
    return params.INCLUDE_INTERNAL ? allTopics : allTopics.findAll { !it.startsWith('_') }
}

def handleEmptyTopics() {
    env.TOPICS_COUNT = '0'
    env.TOPICS_FOUND = ''
    
    writeFile file: env.TOPICS_LIST_FILE, text: "# No topics found\n"
    writeFile file: env.TOPICS_JSON_FILE, text: '{"topics": [], "count": 0, "timestamp": "' + new Date().format('yyyy-MM-dd HH:mm:ss') + '"}'
    writeFile file: env.TOPICS_CSV_FILE, text: "topic_name\n"
}

def saveTopicsAsArtifacts(topics) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Text format
    def textContent = "# Kafka Topics List\n# Generated: ${timestamp}\n# Total topics: ${topics.size()}\n\n"
    textContent += topics.join('\n')
    writeFile file: env.TOPICS_LIST_FILE, text: textContent
    
    // JSON format
    def jsonData = [
        metadata: [
            timestamp: timestamp,
            total_topics: topics.size(),
            include_internal: params.INCLUDE_INTERNAL,
            bootstrap_server: params.KAFKA_BOOTSTRAP_SERVER,
            security_protocol: params.SECURITY_PROTOCOL,
            build_number: env.BUILD_NUMBER,
            job_name: env.JOB_NAME
        ],
        topics: topics
    ]
    writeJSON file: env.TOPICS_JSON_FILE, json: jsonData, pretty: 2
    
    // CSV format
    def csvContent = "topic_name,topic_index\n"
    topics.eachWithIndex { topic, index ->
        csvContent += "${topic},${index + 1}\n"
    }
    writeFile file: env.TOPICS_CSV_FILE, text: csvContent
    
    echo "💾 Topics saved in multiple formats"
}

def generateSummaryReport() {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def buildUrl = env.BUILD_URL ?: 'N/A'
    def buildNumber = env.BUILD_NUMBER ?: 'N/A'
    
    def report = """# Kafka Topics Summary Report

## Build Information
- **Build Number**: ${buildNumber}
- **Build URL**: ${buildUrl}
- **Timestamp**: ${timestamp}
- **Jenkins Job**: ${env.JOB_NAME ?: 'N/A'}

## Configuration
- **Compose Directory**: ${params.COMPOSE_DIR}
- **Bootstrap Server**: ${params.KAFKA_BOOTSTRAP_SERVER}
- **Security Protocol**: ${params.SECURITY_PROTOCOL}
- **Include Internal Topics**: ${params.INCLUDE_INTERNAL ? 'Yes' : 'No'}
- **Output Format**: ${params.OUTPUT_FORMAT}

## Results
- **Total Topics Found**: ${env.TOPICS_COUNT}
- **Topics List**: ${env.TOPICS_FOUND ?: 'None'}

## Artifact Files Generated
- **Text List**: ${env.TOPICS_LIST_FILE}
- **JSON Format**: ${env.TOPICS_JSON_FILE}
- **CSV Format**: ${env.TOPICS_CSV_FILE}
- **This Report**: ${env.TOPICS_REPORT_FILE}

## Security
- Credentials managed through Jenkins Secrets
- Client configuration created securely in container
- No sensitive data exposed in logs

## Usage in Other Pipelines
```groovy
// Copy artifacts from this build
copyArtifacts(
    projectName: '${env.JOB_NAME}',
    selector: lastSuccessful(),
    filter: '${env.TOPICS_LIST_FILE},${env.TOPICS_JSON_FILE}'
)

// Read topics as list
def topicsList = readFile('${env.TOPICS_LIST_FILE}').split('\\\\n').findAll { it.trim() && !it.startsWith('#') }

// Read topics as JSON
def topicsJson = readJSON file: '${env.TOPICS_JSON_FILE}'
def topics = topicsJson.topics
```
"""
    
    writeFile file: env.TOPICS_REPORT_FILE, text: report
}

def printUsageInstructions() {
    echo """
🔗 To use these artifacts in other pipelines:
   copyArtifacts(projectName: '${env.JOB_NAME}', selector: lastSuccessful(), filter: '*.txt,*.json,*.csv')
"""
}

def cleanupClientConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
        echo "🧹 Client configuration cleaned up"
    } catch (Exception e) {
        echo "⚠️ Could not cleanup client config (container may not be running)"
    }
}