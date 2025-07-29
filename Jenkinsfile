properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name to produce messages to'),
        text(name: 'MESSAGE_DATA', defaultValue: '{"message": "Hello World", "timestamp": "2024-01-01T00:00:00Z"}', description: 'Message data (JSON format for single message, or multiple lines for multiple messages)'),
        string(name: 'MESSAGE_COUNT', defaultValue: '1', description: 'Number of messages to produce (will repeat the message data)'),
        booleanParam(name: 'USE_FILE_INPUT', defaultValue: false, description: 'Use file input instead of parameter data'),
        string(name: 'INPUT_FILE_PATH', defaultValue: '/tmp/input-messages.json', description: 'Path to input file (only used when USE_FILE_INPUT is true)')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
        PRODUCER_OUTPUT_FILE = 'producer-results.txt'
        MESSAGE_DATA_FILE = '/tmp/producer-messages.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME parameter is required to produce messages.")
                    }

                    if (params.MESSAGE_COUNT && !params.MESSAGE_COUNT.isNumber()) {
                        error("❌ MESSAGE_COUNT must be a valid number")
                    }

                    def messageCount = params.MESSAGE_COUNT.toInteger()
                    if (messageCount <= 0 || messageCount > 10000) {
                        error("❌ MESSAGE_COUNT must be between 1 and 10000")
                    }

                    echo "✅ Parameters validated successfully"
                    echo "   Topic: ${params.TOPIC_NAME}"
                    echo "   Mode: Simple String Producer"
                    echo "   Message Count: ${messageCount}"
                }
            }
        }

        stage('Create Kafka Client Config') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Prepare Message Data') {
            steps {
                script {
                    if (params.USE_FILE_INPUT) {
                        echo "📁 Using file input: ${params.INPUT_FILE_PATH}"
                        prepareMessageDataFromFile()
                    } else {
                        echo "📝 Using parameter input data"
                        prepareMessageDataFromParameter()
                    }
                }
            }
        }

        stage('Produce Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        def topicName = params.TOPIC_NAME.trim()
                        echo "🚀 Producing messages to topic: ${topicName}"
                        def result = produceKafkaMessages(topicName, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                        echo result
                        saveProducerResults(result)
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupClientConfig()
            }
        }
        success {
            script {
                archiveArtifacts artifacts: "${env.PRODUCER_OUTPUT_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Producer results archived successfully."
                echo "✅ Message production completed successfully!"
            }
        }
        failure {
            script {
                echo "❌ Message production failed. Check the logs above for details."
            }
        }
    }
}

def produceKafkaMessages(topicName, username, password) {
    try {
        def produceOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    
                    # Create producer configuration
                    cat > /tmp/producer.properties << "PRODUCER_EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
PRODUCER_EOF
                    
                    echo "Producer configuration:"
                    cat /tmp/producer.properties
                    echo ""
                    
                    echo "Message data preview:"
                    head -3 ${env.MESSAGE_DATA_FILE}
                    echo ""
                    
                    MESSAGE_COUNT=\$(wc -l < ${env.MESSAGE_DATA_FILE})
                    echo "Producing \$MESSAGE_COUNT messages to topic ${topicName}..."
                    
                    START_TIME=\$(date +%s)
                    kafka-console-producer --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --topic "${topicName}" \\
                        --producer.config /tmp/producer.properties < ${env.MESSAGE_DATA_FILE}
                    END_TIME=\$(date +%s)
                    
                    DURATION=\$((END_TIME - START_TIME))
                    echo ""
                    echo "✅ Successfully produced \$MESSAGE_COUNT messages in \$DURATION seconds"
                    echo "   Topic: ${topicName}"
                    echo "   Producer Mode: Simple String Producer"
                    echo "   Serializer: StringSerializer"
                '
            """,
            returnStdout: true
        ).trim()

        return "✅ Messages produced successfully.\n${produceOutput}"

    } catch (Exception e) {
        return "ERROR: Failed to produce messages to topic '${topicName}' - ${e.getMessage()}"
    }
}

def createKafkaClientConfig(username, password) {
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
        default:
            securityConfig = """
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            break
    }

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
${securityConfig}
EOF'
    """
}

def prepareMessageDataFromFile() {
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            if [ -f "${params.INPUT_FILE_PATH}" ]; then
                cp "${params.INPUT_FILE_PATH}" "${env.MESSAGE_DATA_FILE}"
                echo "✅ Message data copied from ${params.INPUT_FILE_PATH}"
            else
                echo "❌ Input file ${params.INPUT_FILE_PATH} not found"
                exit 1
            fi
        '
    """
}

def prepareMessageDataFromParameter() {
    def messageCount = params.MESSAGE_COUNT.toInteger()
    def messageData = params.MESSAGE_DATA.trim()
    
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            echo "Preparing ${messageCount} messages..."
            rm -f "${env.MESSAGE_DATA_FILE}"
            for i in \$(seq 1 ${messageCount}); do
                echo "${messageData}" >> "${env.MESSAGE_DATA_FILE}"
            done
            echo "✅ Message data file prepared with ${messageCount} messages"
            echo "Sample content:"
            head -3 "${env.MESSAGE_DATA_FILE}"
        '
    """
}

def saveProducerResults(result) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Kafka Message Producer Results (Simple String Producer)
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Producer Mode: Simple String Producer
# Message Count: ${params.MESSAGE_COUNT}
# Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security Protocol: ${params.SECURITY_PROTOCOL}

================================================================================
PRODUCER EXECUTION RESULTS
================================================================================

${result}

================================================================================
CONFIGURATION SUMMARY
================================================================================

Topic Name: ${params.TOPIC_NAME}
Producer Mode: Simple String Producer
Message Count: ${params.MESSAGE_COUNT}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}
Use File Input: ${params.USE_FILE_INPUT}
Input File Path: ${params.INPUT_FILE_PATH}

================================================================================
"""

    writeFile file: env.PRODUCER_OUTPUT_FILE, text: content
}

def cleanupClientConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE} ${env.MESSAGE_DATA_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}