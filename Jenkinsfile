properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: '', description: 'Consumer group ID (optional - auto-generated if empty)'),
        string(name: 'MAX_MESSAGES', defaultValue: '10', description: 'Max messages to consume (defaults to 10)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], description: 'Where to start consuming (defaults to latest)'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL', 'PLAINTEXT'], description: 'Security protocol (defaults to SASL_PLAINTEXT)'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: '', description: 'Kafka bootstrap server (optional - defaults to broker:29093)'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: '', description: 'Schema Registry URL (optional - defaults to http://schema-registry:8081)'),
        string(name: 'COMPOSE_DIR', defaultValue: '', description: 'Docker compose directory (optional - defaults to /confluent/cp-mysetup/cp-all-in-one)'),
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds (defaults to 30)'),
        string(name: 'SCHEMA_REGISTRY_CONTAINER', defaultValue: '', description: 'Schema Registry container name (optional - defaults to schema-registry)')
    ])
])

pipeline {
    agent any
    
    environment {
        CLIENT_CONFIG_FILE = '/tmp/avro-consumer.properties'
        MESSAGES_FILE = 'consumed-avro-messages.txt'
        STATS_FILE = 'consumption-stats.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    // Set environment variables with smart defaults
                    env.COMPOSE_DIR = params.COMPOSE_DIR?.trim() ?: '/confluent/cp-mysetup/cp-all-in-one'
                    env.KAFKA_SERVER = params.KAFKA_BOOTSTRAP_SERVER?.trim() ?: 'broker:29093'
                    env.SCHEMA_REGISTRY_URL = params.SCHEMA_REGISTRY_URL?.trim() ?: 'http://schema-registry:8081'
                    env.TIMEOUT_SECONDS = params.TIMEOUT_SECONDS?.trim() ?: '30'
                    env.SCHEMA_REGISTRY_CONTAINER = params.SCHEMA_REGISTRY_CONTAINER?.trim() ?: 'schema-registry'
                    env.SECURITY_PROTOCOL = params.SECURITY_PROTOCOL?.trim() ?: 'SASL_PLAINTEXT'
                    env.OFFSET_RESET = params.OFFSET_RESET?.trim() ?: 'latest'
                    
                    // Generate consumer group if not provided
                    if (!params.CONSUMER_GROUP_ID?.trim()) {
                        env.CONSUMER_GROUP_ID = "jenkins-avro-consumer-${System.currentTimeMillis()}"
                        echo "🔄 Auto-generated Consumer Group: ${env.CONSUMER_GROUP_ID}"
                    } else {
                        env.CONSUMER_GROUP_ID = params.CONSUMER_GROUP_ID.trim()
                    }
                    
                    // Handle max messages - default to 10 if not specified
                    if (!params.MAX_MESSAGES?.trim()) {
                        env.MAX_MESSAGES = '10'
                        echo "📝 Max Messages: ${env.MAX_MESSAGES} (default)"
                    } else {
                        env.MAX_MESSAGES = params.MAX_MESSAGES.trim()
                        echo "📝 Max Messages: ${env.MAX_MESSAGES}"
                    }
                    
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    
                    echo "✅ Topic: ${params.TOPIC_NAME}"
                    echo "📊 Consumer Group: ${env.CONSUMER_GROUP_ID}"
                    echo "⏰ Timeout: ${env.TIMEOUT_SECONDS}s"
                    echo "🔒 Security Protocol: ${env.SECURITY_PROTOCOL}"
                    echo "📍 Offset Reset: ${env.OFFSET_RESET}"
                    echo "🏠 Compose Dir: ${env.COMPOSE_DIR}"
                    echo "🌐 Kafka Server: ${env.KAFKA_SERVER}"
                    echo "🔗 Schema Registry: ${env.SCHEMA_REGISTRY_URL}"
                }
            }
        }

        stage('Setup Client Config') {
            steps {
                script {
                    if (env.SECURITY_PROTOCOL in ['SASL_PLAINTEXT', 'SASL_SSL']) {
                        withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                                       usernameVariable: 'KAFKA_USER', 
                                                       passwordVariable: 'KAFKA_PASS')]) {
                            createKafkaClientConfig(env.KAFKA_USER, env.KAFKA_PASS)
                        }
                    } else {
                        // For PLAINTEXT, create config without credentials
                        createKafkaClientConfig('', '')
                    }
                }
            }
        }

        stage('Consume Avro Messages') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    def messages = consumeAvroMessages()
                    def endTime = System.currentTimeMillis()
                    def duration = endTime - startTime
                    
                    saveMessages(messages, duration)
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: "${env.MESSAGES_FILE}, ${env.STATS_FILE}", allowEmptyArchive: true
            echo "✅ Avro message consumption completed!"
        }
        failure {
            echo "❌ Avro message consumption failed"
        }
        always {
            script {
                cleanupClientConfig()
            }
        }
    }
}

def consumeAvroMessages() {
    def maxMsgs = env.MAX_MESSAGES.toInteger()
    def timeoutSeconds = env.TIMEOUT_SECONDS.toInteger()
    def composeDir = env.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER
    def schemaRegistryUrl = env.SCHEMA_REGISTRY_URL
    def schemaRegistryContainer = env.SCHEMA_REGISTRY_CONTAINER
    def offsetFlag = env.OFFSET_RESET == 'earliest' ? '--from-beginning' : ''
    def topicName = params.TOPIC_NAME
    
    if (env.SECURITY_PROTOCOL in ['SASL_PLAINTEXT', 'SASL_SSL']) {
        withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                         usernameVariable: 'KAFKA_USER', 
                                         passwordVariable: 'KAFKA_PASS')]) {
            
            return executeConsumerWithTimeout(composeDir, schemaRegistryContainer, timeoutSeconds, 
                                            kafkaServer, schemaRegistryUrl, offsetFlag, maxMsgs, 
                                            topicName, env.KAFKA_USER, env.KAFKA_PASS)
        }
    } else {
        return executeConsumerWithTimeout(composeDir, schemaRegistryContainer, timeoutSeconds, 
                                        kafkaServer, schemaRegistryUrl, offsetFlag, maxMsgs, 
                                        topicName, '', '')
    }
}

def executeConsumerWithTimeout(composeDir, schemaRegistryContainer, timeoutSeconds, kafkaServer, 
                              schemaRegistryUrl, offsetFlag, maxMsgs, topicName, username, password) {
    
    // Build security properties
    def securityProps = buildSecurityProperties(username, password)
    
    // Method 1: Use timeout with max-messages
    def result = sh(
        script: """
            set -e
            echo "🚀 Starting Avro consumer for topic: ${topicName}"
            echo "📊 Max messages: ${maxMsgs}, Timeout: ${timeoutSeconds}s"
            
            # Use timeout command to limit execution time
            timeout ${timeoutSeconds}s docker exec ${schemaRegistryContainer} kafka-avro-console-consumer \\
                --bootstrap-server ${kafkaServer} \\
                --topic ${topicName} \\
                --max-messages ${maxMsgs} \\
                --from-beginning \\
                --property schema.registry.url=${schemaRegistryUrl} \\
                --group ${env.CONSUMER_GROUP_ID} \\
                ${securityProps} \\
                2>/dev/null | grep '^{' || echo "No messages found"
            
            echo "✅ Consumer finished"
        """,
        returnStdout: true
    )

    return result.trim()
}

def buildSecurityProperties(username, password) {
    switch(env.SECURITY_PROTOCOL) {
        case 'SASL_PLAINTEXT':
        case 'SASL_SSL':
            if (username && password) {
                return """--consumer-property security.protocol=${env.SECURITY_PROTOCOL} \\
                            --consumer-property sasl.mechanism=PLAIN \\
                            --consumer-property sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";'"""
            } else {
                echo "⚠️ SASL protocol selected but no credentials provided, using PLAINTEXT"
                return "--consumer-property security.protocol=PLAINTEXT"
            }
        case 'PLAINTEXT':
            return "--consumer-property security.protocol=PLAINTEXT"
        default:
            return "--consumer-property security.protocol=PLAINTEXT"
    }
}

def cleanupClientConfig() {
    try {
        def composeDir = env.COMPOSE_DIR
        def schemaRegistryContainer = env.SCHEMA_REGISTRY_CONTAINER
        sh """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
            exec -T ${schemaRegistryContainer} bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
        echo "⚠️ Cleanup warning: ${e.message}"
    }
}

def createKafkaClientConfig(username, password) {
    def composeDir = env.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER
    def schemaRegistryContainer = env.SCHEMA_REGISTRY_CONTAINER
    
    def securityConfig = ""
    switch(env.SECURITY_PROTOCOL) {
        case 'SASL_PLAINTEXT':
        case 'SASL_SSL':
            if (username && password) {
                securityConfig = """
security.protocol=${env.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            } else {
                securityConfig = """
security.protocol=PLAINTEXT
"""
            }
            break
        case 'PLAINTEXT':
            securityConfig = """
security.protocol=PLAINTEXT
"""
            break
        default:
            securityConfig = """
security.protocol=PLAINTEXT
"""
            break
    }
    sh """
        docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
        exec -T ${schemaRegistryContainer} bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${kafkaServer}
${securityConfig}
EOF'
    """
}

def saveMessages(messages, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Filter out system messages and keep only JSON messages
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('Consumer finished') &&
                   !trimmed.contains('WARN') &&
                   !trimmed.contains('ERROR') &&
                   !trimmed.contains('Starting Avro consumer') &&
                   !trimmed.contains('Max messages:') &&
                   !trimmed.startsWith('#') &&
                   !trimmed.startsWith('🚀') &&
                   !trimmed.startsWith('📊') &&
                   !trimmed.startsWith('✅') &&
                   (trimmed.startsWith('{') || trimmed.contains('|')) // JSON or has our key-value separator
        }
    
    def messageCount = messageLines.size()
    
    def content = """# Avro Kafka Consumer Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${env.CONSUMER_GROUP_ID}
# Messages Retrieved: ${messageCount}
# Max Messages Requested: ${env.MAX_MESSAGES}
# Duration: ${duration}ms
# Offset Reset: ${env.OFFSET_RESET}
# Schema Registry: ${env.SCHEMA_REGISTRY_URL}
# Timeout: ${env.TIMEOUT_SECONDS}s

"""

    if (messageCount == 0) {
        content += """No Avro messages found.

Possible reasons:
- Topic is empty
- No new messages since last consumption (if using 'latest' offset)
- Messages already consumed by this consumer group
- Consumer timeout reached before any messages arrived
- Schema Registry connection issues
- Topic contains non-Avro messages

Try using 'earliest' offset to read from the beginning of the topic.

"""
    } else {
        content += """${'='*60}
AVRO MESSAGES (${messageCount}/${env.MAX_MESSAGES})
${'='*60}

"""
        messageLines.eachWithIndex { message, index ->
            // Check if message has timestamp and key separator
            if (message.contains(' | ')) {
                def parts = message.split(' \\| ', 3)
                if (parts.length >= 3) {
                    content += """[${index + 1}] ${parts[0]}
Key: ${parts[1] == 'null' ? '(no key)' : parts[1]}
Value: ${parts[2]}

"""
                } else {
                    content += "[${index + 1}] ${message}\n\n"
                }
            } else {
                // Assume it's a JSON message without timestamp/key
                content += "[${index + 1}] ${message}\n\n"
            }
        }
        
        if (messageCount == env.MAX_MESSAGES.toInteger()) {
            content += "\n⚠️ Maximum message limit reached. There may be more messages available.\n"
        }
    }
    
    writeFile file: env.MESSAGES_FILE, text: content
    echo "💾 Saved ${messageCount} Avro messages to ${env.MESSAGES_FILE}"
    
    // Create stats file
    def stats = [
        topic: params.TOPIC_NAME,
        consumerGroup: env.CONSUMER_GROUP_ID,
        messageCount: messageCount,
        maxMessages: env.MAX_MESSAGES.toInteger(),
        duration: duration,
        timestamp: timestamp,
        schemaRegistry: env.SCHEMA_REGISTRY_URL,
        offsetReset: env.OFFSET_RESET,
        timeoutSeconds: env.TIMEOUT_SECONDS.toInteger()
    ]
    
    writeFile file: env.STATS_FILE, text: groovy.json.JsonOutput.toJson(stats)
    echo "📊 Consumption stats: ${messageCount} messages in ${duration}ms"
}