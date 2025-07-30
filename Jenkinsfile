properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-avro-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], description: 'Where to start consuming'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL', 'PLAINTEXT'], description: 'Security protocol'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'broker:29093', description: 'Kafka bootstrap server'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://schema-registry:8081', description: 'Schema Registry URL'),
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker compose directory'),
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds')
    ])
])

pipeline {
    agent any
    
    environment {
        MESSAGES_FILE = 'consumed-avro-messages.txt'
        STATS_FILE = 'avro-consumption-stats.json'
        CONTAINER_NAME = 'schema-registry'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    // Set environment variables with defaults
                    env.COMPOSE_DIR = params.COMPOSE_DIR ?: '/confluent/cp-mysetup/cp-all-in-one'
                    env.KAFKA_SERVER = params.KAFKA_BOOTSTRAP_SERVER ?: 'broker:29093'
                    env.SCHEMA_REGISTRY_URL = params.SCHEMA_REGISTRY_URL ?: 'http://schema-registry:8081'
                    
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    echo "✅ Topic: ${params.TOPIC_NAME}"
                    echo "📊 Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "⏰ Timeout: ${params.TIMEOUT_SECONDS}s"
                    echo "📝 Max Messages: ${params.MAX_MESSAGES}"
                    echo "🏠 Compose Dir: ${env.COMPOSE_DIR}"
                    echo "🌐 Kafka Server: ${env.KAFKA_SERVER}"
                    echo "📋 Schema Registry: ${env.SCHEMA_REGISTRY_URL}"
                }
            }
        }

        stage('Check Schema Registry') {
            steps {
                script {
                    def registryHealthy = checkSchemaRegistry()
                    if (!registryHealthy) {
                        error("❌ Schema Registry is not accessible at ${env.SCHEMA_REGISTRY_URL}")
                    }
                    echo "✅ Schema Registry verified"
                }
            }
        }

        stage('Check Topic Schema') {
            steps {
                script {
                    def schemaExists = checkTopicSchema()
                    if (!schemaExists) {
                        echo "⚠️ Warning: No schema found for topic '${params.TOPIC_NAME}' (will attempt to consume anyway)"
                    } else {
                        echo "✅ Schema found for topic"
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
                    
                    saveAvroMessages(messages, duration)
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
                // No cleanup needed for avro consumer
                echo "🧹 Cleanup completed"
            }
        }
    }
}

def checkSchemaRegistry() {
    try {
        def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
        def result = sh(
            script: """
                docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
                exec -T ${env.CONTAINER_NAME} bash -c "
                    curl -s ${env.SCHEMA_REGISTRY_URL}/subjects || echo 'FAILED'
                " 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        return !result.contains('FAILED') && !result.isEmpty()
    } catch (Exception e) {
        echo "⚠️ Could not verify Schema Registry: ${e.message}"
        return false
    }
}

def checkTopicSchema() {
    try {
        def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
        def result = sh(
            script: """
                docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
                exec -T ${env.CONTAINER_NAME} bash -c "
                    curl -s ${env.SCHEMA_REGISTRY_URL}/subjects/${params.TOPIC_NAME}-value/versions/latest 2>/dev/null || echo 'NOT_FOUND'
                "
            """,
            returnStdout: true
        ).trim()
        return !result.contains('NOT_FOUND') && !result.contains('Subject not found')
    } catch (Exception e) {
        echo "⚠️ Could not check topic schema: ${e.message}"
        return false
    }
}

def consumeAvroMessages() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    
    def securityProps = buildSecurityProperties()
    
    def result = sh(
        script: """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
            exec -T ${env.CONTAINER_NAME} bash -c '
                # Set environment variables
                export KAFKA_OPTS=""
                export JMX_PORT=""
                export KAFKA_JMX_OPTS=""
                export KAFKA_HEAP_OPTS=""
                
                # Consume Avro messages
                timeout ${timeoutSeconds}s kafka-avro-console-consumer \\
                    --bootstrap-server ${kafkaServer} \\
                    --topic ${params.TOPIC_NAME} \\
                    --from-beginning \\
                    --property schema.registry.url=${env.SCHEMA_REGISTRY_URL} \\
                    --consumer-property group.id=${params.CONSUMER_GROUP_ID} \\
                    --consumer-property auto.offset.reset=${params.OFFSET_RESET} \\
                    --consumer-property enable.auto.commit=true \\
                    --consumer-property auto.commit.interval.ms=1000 \\
                    --consumer-property session.timeout.ms=30000 \\
                    --consumer-property heartbeat.interval.ms=3000 \\
                    ${securityProps} \\
                    ${maxMsgFlag} \\
                    --property print.key=true \\
                    --property print.timestamp=true \\
                    --property key.separator=" | " \\
                    --timeout-ms 10000 2>/dev/null || echo "Avro Consumer finished"
            '
        """,
        returnStdout: true
    )
    
    return result.trim()
}

def buildSecurityProperties() {
    def securityProps = ""
    
    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                     usernameVariable: 'KAFKA_USER', 
                                     passwordVariable: 'KAFKA_PASS')]) {
        switch(params.SECURITY_PROTOCOL) {
            case 'SASL_PLAINTEXT':
            case 'SASL_SSL':
                securityProps = """--consumer-property security.protocol=${params.SECURITY_PROTOCOL} \\
                    --consumer-property sasl.mechanism=PLAIN \\
                    --consumer-property sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";'"""
                break
            case 'PLAINTEXT':
                securityProps = "--consumer-property security.protocol=PLAINTEXT"
                break
            default:
                securityProps = """--consumer-property security.protocol=SASL_PLAINTEXT \\
                    --consumer-property sasl.mechanism=PLAIN \\
                    --consumer-property sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";'"""
                break
        }
    }
    
    return securityProps
}

def saveAvroMessages(messages, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Filter out system messages and keep only actual Kafka messages
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('Avro Consumer finished') &&
                   !trimmed.contains('WARN') &&
                   !trimmed.contains('ERROR') &&
                   !trimmed.startsWith('#') &&
                   !trimmed.contains('SLF4J') &&
                   trimmed.length() > 0
        }
    
    def messageCount = messageLines.size()
    
    def content = """# Kafka Avro Consumer Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Schema Registry: ${env.SCHEMA_REGISTRY_URL}
# Messages Retrieved: ${messageCount}
# Duration: ${duration}ms
# Offset Reset: ${params.OFFSET_RESET}
# Security Protocol: ${params.SECURITY_PROTOCOL}

"""

    if (messageCount == 0) {
        content += """No Avro messages found.

Possible reasons:
- Topic is empty or contains no Avro messages
- Messages already consumed by this consumer group
- Offset setting (try 'earliest' to read from beginning)
- Consumer timeout reached
- Schema incompatibility issues
- Topic doesn't use Avro serialization

"""
    } else {
        content += """${'='*60}
AVRO MESSAGES
${'='*60}

"""
        messageLines.eachWithIndex { message, index ->
            // Check if message has the key separator format
            if (message.contains(' | ')) {
                def parts = message.split(' \\| ', 3)
                if (parts.length >= 3) {
                    content += """[${index + 1}] ${parts[0]}
Key: ${parts[1] == 'null' ? '(no key)' : parts[1]}
Avro Value: ${parts[2]}

"""
                } else {
                    content += "[${index + 1}] ${message}\n\n"
                }
            } else {
                // Message without separator, likely just the value
                content += """[${index + 1}] 
Avro Value: ${message}

"""
            }
        }
    }
    
    // Generate statistics
    def stats = [
        timestamp: timestamp,
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        schemaRegistry: env.SCHEMA_REGISTRY_URL,
        messageCount: messageCount,
        durationMs: duration,
        offsetReset: params.OFFSET_RESET,
        securityProtocol: params.SECURITY_PROTOCOL,
        maxMessages: params.MAX_MESSAGES,
        timeoutSeconds: params.TIMEOUT_SECONDS
    ]
    
    writeFile file: env.MESSAGES_FILE, text: content
    writeFile file: env.STATS_FILE, text: groovy.json.JsonBuilder(stats).toPrettyString()
    
    echo "📊 Saved ${messageCount} Avro messages to ${env.MESSAGES_FILE}"
    echo "📈 Statistics saved to ${env.STATS_FILE}"
}