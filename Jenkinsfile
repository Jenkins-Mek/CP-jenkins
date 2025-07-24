//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name to consume messages from'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-consumer-group', description: 'Consumer group ID'),
        choice(name: 'CONSUMER_MODE', choices: ['WITHOUT_SCHEMA', 'WITH_JSON_SCHEMA', 'WITH_AVRO_SCHEMA', 'WITH_PROTOBUF_SCHEMA'], description: 'Message consumption mode'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://schema-registry:8081', description: 'Schema Registry URL (only used when schema is enabled)'),
        choice(name: 'OFFSET_RESET', choices: ['earliest', 'latest', 'none'], defaultValue: 'latest', description: 'Auto offset reset strategy'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Maximum number of messages to consume (0 for unlimited)'),
        string(name: 'TIMEOUT_MS', defaultValue: '30000', description: 'Consumer timeout in milliseconds'),
        booleanParam(name: 'SAVE_TO_FILE', defaultValue: true, description: 'Save consumed messages to file'),
        string(name: 'OUTPUT_FILE_PATH', defaultValue: '/tmp/consumed-messages.json', description: 'Path to save consumed messages (only used when SAVE_TO_FILE is true)'),
        booleanParam(name: 'SHOW_KEY', defaultValue: true, description: 'Display message keys in output'),
        booleanParam(name: 'SHOW_HEADERS', defaultValue: false, description: 'Display message headers in output'),
        booleanParam(name: 'SHOW_TIMESTAMP', defaultValue: true, description: 'Display message timestamps in output'),
        booleanParam(name: 'SHOW_PARTITION', defaultValue: true, description: 'Display partition information in output')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
        CONSUMER_OUTPUT_FILE = 'consumer-results.txt'
        CONSUMED_MESSAGES_FILE = '/tmp/consumed-messages.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME parameter is required to consume messages.")
                    }
                    
                    if (!params.CONSUMER_GROUP_ID?.trim()) {
                        error("❌ CONSUMER_GROUP_ID parameter is required.")
                    }
                    
                    if (params.MAX_MESSAGES && !params.MAX_MESSAGES.isNumber()) {
                        error("❌ MAX_MESSAGES must be a valid number")
                    }
                    
                    if (params.TIMEOUT_MS && !params.TIMEOUT_MS.isNumber()) {
                        error("❌ TIMEOUT_MS must be a valid number")
                    }
                    
                    def maxMessages = params.MAX_MESSAGES.toInteger()
                    if (maxMessages < 0 || maxMessages > 50000) {
                        error("❌ MAX_MESSAGES must be between 0 and 50000 (0 for unlimited)")
                    }
                    
                    echo "✅ Parameters validated successfully"
                    echo "   Topic: ${params.TOPIC_NAME}"
                    echo "   Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "   Mode: ${params.CONSUMER_MODE}"
                    echo "   Max Messages: ${maxMessages == 0 ? 'Unlimited' : maxMessages}"
                    echo "   Offset Reset: ${params.OFFSET_RESET}"
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

        stage('Validate Schema Registry') {
            when {
                expression { params.CONSUMER_MODE != 'WITHOUT_SCHEMA' }
            }
            steps {
                script {
                    echo "🔍 Validating Schema Registry connection..."
                    validateSchemaRegistry()
                }
            }
        }

        stage('Check Topic Exists') {
            steps {
                script {
                    echo "🔍 Checking if topic exists..."
                    checkTopicExists()
                }
            }
        }

        stage('Consume Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        def topicName = params.TOPIC_NAME.trim()
                        echo "🔽 Consuming messages from topic: ${topicName}"
                        def result = consumeKafkaMessages(topicName, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                        echo result
                        saveConsumerResults(result)
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
                archiveArtifacts artifacts: "${env.CONSUMER_OUTPUT_FILE}", fingerprint: true, allowEmptyArchive: true
                if (params.SAVE_TO_FILE) {
                    archiveArtifacts artifacts: "consumed-messages-*.json", fingerprint: true, allowEmptyArchive: true
                }
                echo "📦 Consumer results archived successfully."
                echo "✅ Message consumption completed successfully!"
            }
        }
        failure {
            script {
                echo "❌ Message consumption failed. Check the logs above for details."
            }
        }
    }
}

def consumeKafkaMessages(topicName, username, password) {
    try {
        // Determine deserializer based on consumer mode
        def valueDeserializer = getValueDeserializer(params.CONSUMER_MODE)
        def maxMessages = params.MAX_MESSAGES.toInteger()
        def maxMessagesFlag = maxMessages > 0 ? "--max-messages ${maxMessages}" : ""
        
        // Build formatter options
        def formatterOptions = buildFormatterOptions()
        
        def consumeOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    
                    # Create consumer configuration
                    cat > /tmp/consumer.properties << "CONSUMER_EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
group.id=${params.CONSUMER_GROUP_ID}
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=${valueDeserializer}
auto.offset.reset=${params.OFFSET_RESET}
enable.auto.commit=true
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
consumer.timeout.ms=${params.TIMEOUT_MS}
${getSchemaRegistryConfig()}
CONSUMER_EOF
                    
                    echo "Consumer configuration created:"
                    echo "Bootstrap servers: ${params.KAFKA_BOOTSTRAP_SERVER}"
                    echo "Group ID: ${params.CONSUMER_GROUP_ID}"
                    echo "Value deserializer: ${valueDeserializer}"
                    echo "Auto offset reset: ${params.OFFSET_RESET}"
                    echo ""
                    
                    echo "Starting message consumption from topic ${topicName}..."
                    echo "Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "Offset Reset: ${params.OFFSET_RESET}"
                    echo "Max Messages: ${maxMessages == 0 ? 'Unlimited' : maxMessages}"
                    echo "Timeout: ${params.TIMEOUT_MS}ms"
                    echo ""
                    
                    START_TIME=\$(date +%s)
                    
                    # Create output file for consumed messages
                    CONSUMED_FILE="/tmp/consumed-messages-\$(date +%Y%m%d-%H%M%S).json"
                    
                    # Test consumer connectivity first
                    echo "Testing consumer connectivity..."
                    if ! kafka-console-consumer --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --consumer.config /tmp/consumer.properties \\
                        --topic "${topicName}" \\
                        --timeout-ms 5000 \\
                        --max-messages 1 > /dev/null 2>/tmp/consumer_test.log; then
                        echo "❌ Consumer connectivity test failed:"
                        cat /tmp/consumer_test.log
                        exit 1
                    fi
                    echo "✅ Consumer connectivity test passed"
                    
                    # Consume messages with proper timeout handling
                    echo "Starting actual message consumption..."
                    timeout ${params.TIMEOUT_MS.toInteger() / 1000 + 10}s kafka-console-consumer \\
                        --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --topic "${topicName}" \\
                        --consumer.config /tmp/consumer.properties \\
                        ${maxMessagesFlag} \\
                        ${formatterOptions} > "\$CONSUMED_FILE" 2>/tmp/consumer_error.log || {
                            EXIT_CODE=\$?
                            if [ \$EXIT_CODE -eq 124 ]; then
                                echo "⏱️  Consumer timed out after ${params.TIMEOUT_MS}ms"
                            elif [ \$EXIT_CODE -eq 1 ]; then
                                echo "✅ Consumer finished normally (no more messages or reached max messages)"
                            else
                                echo "❌ Consumer exited with code \$EXIT_CODE"
                                if [ -s /tmp/consumer_error.log ]; then
                                    echo "Error details:"
                                    cat /tmp/consumer_error.log
                                fi
                            fi
                        }
                    
                    END_TIME=\$(date +%s)
                    DURATION=\$((END_TIME - START_TIME))
                    
                    # Count consumed messages
                    MESSAGE_COUNT=\$(wc -l < "\$CONSUMED_FILE" 2>/dev/null || echo "0")
                    
                    echo ""
                    echo "✅ Consumption completed in \$DURATION seconds"
                    echo "   Topic: ${topicName}"
                    echo "   Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "   Messages Consumed: \$MESSAGE_COUNT"
                    echo "   Consumer Mode: ${params.CONSUMER_MODE}"
                    echo "   Deserializer: ${valueDeserializer}"
                    
                    # Show sample messages if any were consumed
                    if [ \$MESSAGE_COUNT -gt 0 ]; then
                        echo ""
                        echo "📝 Sample consumed messages (first 5):"
                        head -5 "\$CONSUMED_FILE" 2>/dev/null || echo "No messages to display"
                        
                        # Copy to workspace if SAVE_TO_FILE is enabled
                        if [ "${params.SAVE_TO_FILE}" = "true" ]; then
                            cp "\$CONSUMED_FILE" "${env.CONSUMED_MESSAGES_FILE}"
                            echo ""
                            echo "💾 Messages saved to: \$CONSUMED_FILE"
                        fi
                    else
                        echo ""
                        echo "ℹ️  No messages were consumed from the topic"
                        echo "This could mean:"
                        echo "   - The topic is empty"
                        echo "   - All messages were consumed by other consumers"
                        echo "   - The offset reset strategy skipped available messages"
                        
                        # Show consumer group status
                        echo ""
                        echo "Consumer group status:"
                        kafka-consumer-groups --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                            --command-config /tmp/consumer.properties \\
                            --describe --group ${params.CONSUMER_GROUP_ID} 2>/dev/null || echo "Consumer group details not available"
                    fi
                '
            """,
            returnStdout: true
        ).trim()

        return "✅ Messages consumed successfully.\n${consumeOutput}"

    } catch (Exception e) {
        return "ERROR: Failed to consume messages from topic '${topicName}' - ${e.getMessage()}"
    }
}

def getValueDeserializer(consumerMode) {
    switch (consumerMode.toLowerCase()) {
        case 'with_json_schema':
            return 'io.confluent.kafka.serializers.json.KafkaJsonSchemaDeserializer'
        case 'with_avro_schema':
            return 'io.confluent.kafka.serializers.KafkaAvroDeserializer'
        case 'with_protobuf_schema':
            return 'io.confluent.kafka.serializers.protobuf.KafkaProtobufDeserializer'
        default:
            return 'org.apache.kafka.common.serialization.StringDeserializer'
    }
}

def buildFormatterOptions() {
    def options = []
    
    if (params.SHOW_KEY) {
        options.add("--property print.key=true")
        options.add("--property key.separator=\" | \"")
    }
    
    if (params.SHOW_TIMESTAMP) {
        options.add("--property print.timestamp=true")
    }
    
    if (params.SHOW_PARTITION) {
        options.add("--property print.partition=true")
    }
    
    if (params.SHOW_HEADERS) {
        options.add("--property print.headers=true")
    }
    
    return options.join(" ")
}

def getSchemaRegistryConfig() {
    if (params.CONSUMER_MODE != 'WITHOUT_SCHEMA') {
        return "schema.registry.url=${params.SCHEMA_REGISTRY_URL}"
    }
    return ""
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

def checkTopicExists() {
    try {
        withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
            sh """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    
                    echo "🔍 Checking Kafka connection and topic existence..."
                    echo "Bootstrap server: ${params.KAFKA_BOOTSTRAP_SERVER}"
                    echo "Topic: ${params.TOPIC_NAME}"
                    
                    # First, test basic connectivity by listing all topics
                    echo "Testing Kafka connectivity..."
                    if kafka-topics --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --command-config ${env.CLIENT_CONFIG_FILE} \\
                        --list > /tmp/topic_list.txt 2>&1; then
                        echo "✅ Successfully connected to Kafka cluster"
                        echo "Available topics:"
                        cat /tmp/topic_list.txt | head -10
                        
                        # Check if our specific topic exists in the list
                        if grep -q "^${params.TOPIC_NAME}$" /tmp/topic_list.txt; then
                            echo "✅ Topic ${params.TOPIC_NAME} found in topic list"
                            
                            # Get detailed topic information
                            echo "Getting topic details..."
                            kafka-topics --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                                --command-config ${env.CLIENT_CONFIG_FILE} \\
                                --describe --topic "${params.TOPIC_NAME}"
                        else
                            echo "❌ Topic ${params.TOPIC_NAME} not found in available topics"
                            echo "Available topics are:"
                            cat /tmp/topic_list.txt
                            exit 1
                        fi
                    else
                        echo "❌ Failed to connect to Kafka cluster"
                        echo "Error output:"
                        cat /tmp/topic_list.txt
                        exit 1
                    fi
                '
            """
        }
    } catch (Exception e) {
        error("❌ Topic validation failed: ${e.getMessage()}")
    }
}

def validateSchemaRegistry() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" ${params.SCHEMA_REGISTRY_URL}/subjects 2>/dev/null)
                if [ "\$RESPONSE" = "200" ]; then
                    echo "✅ Schema Registry is accessible at ${params.SCHEMA_REGISTRY_URL}"
                else
                    echo "❌ Schema Registry is not accessible (HTTP \$RESPONSE)"
                    exit 1
                fi
            '
        """
    } catch (Exception e) {
        error("❌ Schema Registry validation failed: ${e.getMessage()}")
    }
}

def saveConsumerResults(result) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Kafka Message Consumer Results
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Consumer Mode: ${params.CONSUMER_MODE}
# Max Messages: ${params.MAX_MESSAGES}
# Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security Protocol: ${params.SECURITY_PROTOCOL}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

================================================================================
CONSUMER EXECUTION RESULTS
================================================================================

${result}

================================================================================
CONFIGURATION SUMMARY
================================================================================

Topic Name: ${params.TOPIC_NAME}
Consumer Group ID: ${params.CONSUMER_GROUP_ID}
Consumer Mode: ${params.CONSUMER_MODE}
Offset Reset Strategy: ${params.OFFSET_RESET}
Max Messages: ${params.MAX_MESSAGES}
Timeout (ms): ${params.TIMEOUT_MS}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}
Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}
Save to File: ${params.SAVE_TO_FILE}
Output File Path: ${params.OUTPUT_FILE_PATH}
Show Key: ${params.SHOW_KEY}
Show Headers: ${params.SHOW_HEADERS}
Show Timestamp: ${params.SHOW_TIMESTAMP}
Show Partition: ${params.SHOW_PARTITION}

================================================================================
"""

    writeFile file: env.CONSUMER_OUTPUT_FILE, text: content
}

def cleanupClientConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE} /tmp/consumed-messages-*.json" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}