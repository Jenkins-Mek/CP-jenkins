import groovy.json.JsonOutput

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name to produce messages to'),
        choice(name: 'MESSAGE_FORMAT', choices: ['STRING', 'JSON'], description: 'Message format for serialization'),
        text(name: 'MESSAGE_DATA', defaultValue: 'Hello World from Kafka Producer!', description: 'Message data to produce'),
        string(name: 'MESSAGE_COUNT', defaultValue: '1', description: 'Number of messages to produce'),
        booleanParam(name: 'USE_FILE_INPUT', defaultValue: false, description: 'Use file input instead of parameter data'),
        string(name: 'INPUT_FILE_PATH', defaultValue: '/tmp/input-messages.txt', description: 'Path to input file (only used when USE_FILE_INPUT is true)'),
        booleanParam(name: 'ADD_TIMESTAMP', defaultValue: false, description: 'Add timestamp to each message'),
        booleanParam(name: 'ADD_MESSAGE_INDEX', defaultValue: false, description: 'Add message index/counter to each message'),
        string(name: 'MESSAGE_KEY', defaultValue: '', description: 'Optional message key (leave empty for null key)')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
        PRODUCER_OUTPUT_FILE = 'simple-producer-results.txt'
        MESSAGE_DATA_FILE = '/tmp/producer-messages.txt'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME parameter is required to produce messages.")
                    }
                    
                    if (!params.MESSAGE_DATA?.trim() && !params.USE_FILE_INPUT) {
                        error("❌ MESSAGE_DATA parameter is required when not using file input.")
                    }
                    
                    if (params.MESSAGE_COUNT && !params.MESSAGE_COUNT.isNumber()) {
                        error("❌ MESSAGE_COUNT must be a valid number")
                    }
                    
                    def messageCount = params.MESSAGE_COUNT.toInteger()
                    if (messageCount <= 0 || messageCount > 50000) {
                        error("❌ MESSAGE_COUNT must be between 1 and 50000")
                    }
                    
                    echo "✅ Parameters validated successfully"
                    echo "   Topic: ${params.TOPIC_NAME}"
                    echo "   Message Format: ${params.MESSAGE_FORMAT}"
                    echo "   Message Count: ${messageCount}"
                    echo "   Use File Input: ${params.USE_FILE_INPUT}"
                    echo "   Add Timestamp: ${params.ADD_TIMESTAMP}"
                    echo "   Add Message Index: ${params.ADD_MESSAGE_INDEX}"
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
                        def result = produceMessages(topicName, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
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
                cleanupFiles()
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

def produceMessages(topicName, username, password) {
    try {
        def keySerializer = "org.apache.kafka.common.serialization.StringSerializer"
        def valueSerializer = "org.apache.kafka.common.serialization.StringSerializer"
        
        def keyOption = ""
        if (params.MESSAGE_KEY?.trim()) {
            keyOption = "--property key.separator=: --property parse.key=true"
        }
        
        def produceOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    
                    # Create producer configuration
                    cat > /tmp/producer.properties << "PRODUCER_EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
key.serializer=${keySerializer}
value.serializer=${valueSerializer}
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
acks=all
retries=3
batch.size=16384
linger.ms=1
buffer.memory=33554432
PRODUCER_EOF

                    echo "Producer configuration:"
                    cat /tmp/producer.properties
                    echo ""

                    echo "Message format: ${params.MESSAGE_FORMAT}"
                    echo "Message key: ${params.MESSAGE_KEY ?: 'null (no key)'}"
                    echo ""

                    echo "Message data preview:"
                    head -3 ${env.MESSAGE_DATA_FILE}
                    echo ""

                    MESSAGE_COUNT=\$(wc -l < ${env.MESSAGE_DATA_FILE})
                    echo "Producing \$MESSAGE_COUNT messages to topic ${topicName}..."

                    START_TIME=\$(date +%s)
                    kafka-console-producer --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --topic "${topicName}" \\
                        --producer.config /tmp/producer.properties \\
                        ${keyOption} < ${env.MESSAGE_DATA_FILE}
                    END_TIME=\$(date +%s)

                    DURATION=\$((END_TIME - START_TIME))
                    RATE=\$(( MESSAGE_COUNT > 0 && DURATION > 0 ? MESSAGE_COUNT / DURATION : MESSAGE_COUNT ))
                    echo ""
                    echo "✅ Successfully produced \$MESSAGE_COUNT messages in \$DURATION seconds"
                    echo "   Topic: ${topicName}"
                    echo "   Rate: \$RATE messages/second"
                    echo "   Message Format: ${params.MESSAGE_FORMAT}"
                    echo "   Message Key: ${params.MESSAGE_KEY ?: 'null'}"
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
                MESSAGE_COUNT=\$(wc -l < "${env.MESSAGE_DATA_FILE}")
                echo "📊 Found \$MESSAGE_COUNT messages in file"
            else
                echo "❌ Input file ${params.INPUT_FILE_PATH} not found"
                exit 1
            fi
        '
    """
}

def prepareMessageDataFromParameter() {
    def messageCount = params.MESSAGE_COUNT.toInteger()
    def baseMessage = params.MESSAGE_DATA.trim()

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            echo "Preparing ${messageCount} messages..."
            rm -f "${env.MESSAGE_DATA_FILE}"

            for i in \$(seq 1 ${messageCount}); do
                MESSAGE="${baseMessage}"

                # Add message index if requested
                if [ "${params.ADD_MESSAGE_INDEX}" = "true" ]; then
                    if [ "${params.MESSAGE_FORMAT}" = "JSON" ]; then
                        # Assume JSON and add index field
                        MESSAGE=\$(echo "${baseMessage}" | sed "s/}/,\\"messageIndex\\":\$i}/")
                    else
                        # For string format, prepend index
                        MESSAGE="[\$i] ${baseMessage}"
                    fi
                fi

                # Add timestamp if requested
                if [ "${params.ADD_TIMESTAMP}" = "true" ]; then
                    TIMESTAMP=\$(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")
                    if [ "${params.MESSAGE_FORMAT}" = "JSON" ]; then
                        # Assume JSON and add timestamp field
                        MESSAGE=\$(echo "\$MESSAGE" | sed "s/}/,\\"timestamp\\": \\"\$TIMESTAMP\\"}/")
                    else
                        # For string format, append timestamp
                        MESSAGE="\$MESSAGE [timestamp: \$TIMESTAMP]"
                    fi
                fi

                # Add message key if specified
                if [ -n "${params.MESSAGE_KEY}" ]; then
                    echo "${params.MESSAGE_KEY}:\$MESSAGE" >> "${env.MESSAGE_DATA_FILE}"
                else
                    echo "\$MESSAGE" >> "${env.MESSAGE_DATA_FILE}"
                fi
            done

            echo "✅ Message data file prepared with ${messageCount} messages"
            echo "Sample content:"
            head -3 "${env.MESSAGE_DATA_FILE}"
        '
    """
}

def saveProducerResults(result) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Kafka Simple Producer Results
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Message Format: ${params.MESSAGE_FORMAT}
# Message Count: ${params.MESSAGE_COUNT}
# Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security Protocol: ${params.SECURITY_PROTOCOL}

================================================================================
PRODUCER EXECUTION RESULTS
================================================================================

${result}

================================================================================
MESSAGE CONFIGURATION
================================================================================

Message Format: ${params.MESSAGE_FORMAT}
Message Key: ${params.MESSAGE_KEY ?: 'null (no key)'}
Add Timestamp: ${params.ADD_TIMESTAMP}
Add Message Index: ${params.ADD_MESSAGE_INDEX}
Use File Input: ${params.USE_FILE_INPUT}
Input File Path: ${params.INPUT_FILE_PATH}

Sample Message Data:
${params.MESSAGE_DATA}

================================================================================
KAFKA CONFIGURATION
================================================================================

Topic Name: ${params.TOPIC_NAME}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}
Compose Directory: ${params.COMPOSE_DIR}

================================================================================
"""

    writeFile file: env.PRODUCER_OUTPUT_FILE, text: content
}

def cleanupFiles() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE} ${env.MESSAGE_DATA_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}