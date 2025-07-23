properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        booleanParam(name: 'INCLUDE_INTERNAL', defaultValue: false, description: 'Include internal Kafka topics (starting with _)'),
        choice(name: 'OUTPUT_FORMAT', choices: ['list', 'json', 'csv'], description: 'Output format for topics list')
    ])
])

pipeline {
    agent any

    environment {
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        TOPICS_JSON_FILE = 'kafka-topics.json'
        TOPICS_CSV_FILE = 'kafka-topics.csv'
        TOPICS_REPORT_FILE = 'kafka-topics-report.txt'
    }

    stages {
        stage('Validate Environment') {
            steps {
                script {
                    echo "🔍 Validating Kafka environment..."
                    
                    // Check if compose directory exists
                    if (!params.COMPOSE_DIR) {
                        error("COMPOSE_DIR parameter is required")
                    }
                    
                    // Check if docker compose file exists
                    def composeFile = "${params.COMPOSE_DIR}/docker-compose.yml"
                    def checkResult = sh(
                        script: "test -f ${composeFile} && echo 'exists' || echo 'missing'",
                        returnStdout: true
                    ).trim()
                    
                    if (checkResult == 'missing') {
                        error("Docker compose file not found at: ${composeFile}")
                    }
                    
                    echo "✅ Environment validation passed"
                }
            }
        }

        stage('Wait for Kafka Services') {
            steps {
                script {
                    echo "⏳ Waiting for Kafka services to be ready..."
                    
                    // Wait for broker container to be running
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
                                echo "✅ Kafka broker is ready"
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
            }
        }

        stage('Create Client Configuration') {
            steps {
                script {
                    echo "🔧 Creating Kafka client configuration..."
                    
                    // Create client properties file inside the broker container
                    sh """
                        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                        exec -T broker bash -c "
                            cat > /tmp/client.properties << 'EOF'
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
security.protocol=PLAINTEXT
EOF
                        "
                    """
                    
                    echo "✅ Client configuration created"
                }
            }
        }

        stage('List Kafka Topics') {
            steps {
                script {
                    echo "📋 Retrieving all Kafka topics..."

                    // Get topics list from Kafka
                    def topicsOutput = sh(
                        script: """
                            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                            exec -T broker bash -c "
                                export KAFKA_OPTS=''
                                export JMX_PORT=''
                                export KAFKA_JMX_OPTS=''
                                unset JMX_PORT
                                unset KAFKA_JMX_OPTS
                                kafka-topics --list --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} --command-config /tmp/client.properties 2>/dev/null
                            "
                        """,
                        returnStdout: true
                    ).trim()

                    // Process topics list
                    def allTopics = topicsOutput.split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') }
                    
                    // Filter internal topics if needed
                    def topics = params.INCLUDE_INTERNAL ? allTopics : allTopics.findAll { !it.startsWith('_') }
                    
                    if (topics.size() > 0) {
                        echo "✅ Found ${topics.size()} topic(s):"
                        topics.eachWithIndex { topic, index ->
                            echo "  ${index + 1}. ${topic}"
                        }
                        
                        // Store for environment variables
                        env.TOPICS_COUNT = topics.size().toString()
                        env.TOPICS_FOUND = topics.join(',')
                        
                        // Save topics based on selected format
                        saveTopicsAsArtifacts(topics, allTopics.size())
                        
                    } else {
                        echo "⚠️  No topics found in the Kafka cluster"
                        env.TOPICS_COUNT = '0'
                        env.TOPICS_FOUND = ''
                        
                        // Create empty artifacts
                        writeFile file: env.TOPICS_LIST_FILE, text: "# No topics found\n"
                        writeFile file: env.TOPICS_JSON_FILE, text: '{"topics": [], "count": 0, "timestamp": "' + new Date().format('yyyy-MM-dd HH:mm:ss') + '"}'
                        writeFile file: env.TOPICS_CSV_FILE, text: "topic_name\n"
                    }
                }
            }
        }

        stage('Generate Summary Report') {
            steps {
                script {
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

## Usage in Other Pipelines
To read the topics list in another pipeline:

```groovy
// Copy artifacts from this build
copyArtifacts(
    projectName: '${env.JOB_NAME}',
    selector: lastSuccessful(),
    filter: '${env.TOPICS_LIST_FILE},${env.TOPICS_JSON_FILE}'
)

// Read topics as list
def topicsList = readFile('${env.TOPICS_LIST_FILE}').split('\\n').findAll { it.trim() && !it.startsWith('#') }

// Read topics as JSON
def topicsJson = readJSON file: '${env.TOPICS_JSON_FILE}'
def topics = topicsJson.topics
```
"""
                    
                    writeFile file: env.TOPICS_REPORT_FILE, text: report
                    echo "📊 Summary report generated"
                }
            }
        }
    }

    post {
        success {
            script {
                echo "✅ Successfully listed all Kafka topics (${env.TOPICS_COUNT} found)"
                
                // Archive all artifacts
                archiveArtifacts artifacts: "${env.TOPICS_LIST_FILE},${env.TOPICS_JSON_FILE},${env.TOPICS_CSV_FILE},${env.TOPICS_REPORT_FILE}",
                               fingerprint: true,
                               allowEmptyArchive: true
                
                echo "📦 Topics artifacts archived and available for downstream pipelines"
                echo """
🔗 To use these artifacts in other pipelines:
   copyArtifacts(projectName: '${env.JOB_NAME}', selector: lastSuccessful(), filter: '*.txt,*.json,*.csv')
"""
            }
        }
        failure {
            echo """❌ Failed to list topics. Troubleshooting steps:
1. Check if Kafka services are running: docker compose ps
2. Verify broker container is accessible
3. Check Kafka broker logs for errors
4. Validate bootstrap server configuration"""
        }
        always {
            echo "🏁 Kafka topics listing operation completed"
        }
    }
}

// Helper function to save topics in different formats
def saveTopicsAsArtifacts(topics, totalTopicsCount) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // 1. Save as simple text list (one topic per line)
    def textContent = "# Kafka Topics List\n# Generated: ${timestamp}\n# Total topics: ${topics.size()}\n\n"
    textContent += topics.join('\n')
    writeFile file: env.TOPICS_LIST_FILE, text: textContent
    
    // 2. Save as JSON
    def jsonData = [
        metadata: [
            timestamp: timestamp,
            total_topics: topics.size(),
            total_including_internal: totalTopicsCount,
            include_internal: params.INCLUDE_INTERNAL,
            bootstrap_server: params.KAFKA_BOOTSTRAP_SERVER,
            build_number: env.BUILD_NUMBER,
            job_name: env.JOB_NAME
        ],
        topics: topics
    ]
    writeJSON file: env.TOPICS_JSON_FILE, json: jsonData, pretty: 2
    
    // 3. Save as CSV
    def csvContent = "topic_name,topic_index\n"
    topics.eachWithIndex { topic, index ->
        csvContent += "${topic},${index + 1}\n"
    }
    writeFile file: env.TOPICS_CSV_FILE, text: csvContent
    
    echo "💾 Topics saved in ${params.OUTPUT_FORMAT} format and additional formats"
}