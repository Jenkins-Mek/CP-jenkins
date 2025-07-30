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
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds'),
        string(name: 'SCHEMA_REGISTRY_CONTAINER', defaultValue: 'schema-registry', description: 'Schema Registry container name')
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
                    echo "🔗 Schema Registry: ${env.SCHEMA_REGISTRY_URL}"
                }
            }
        }

        stage('Setup Client Config') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                                   usernameVariable: 'KAFKA_USER', 
                                                   passwordVariable: 'KAFKA_PASS')]) {
                        // Create client configuration first
                        createKafkaClientConfig(env.KAFKA_USER, env.KAFKA_PASS)
                    }
                }
            }
        }

        stage('Check Topic') {
            steps {
                script {
                    def topicExists = checkTopicExists()
                    if (!topicExists) {
                        error("❌ Topic '${params.TOPIC_NAME}' does not exist")
                    }
                    echo "✅ Topic verified"
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

def checkTopicExists() {
    try {
        def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
        def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
        def schemaRegistryContainer = params.SCHEMA_REGISTRY_CONTAINER ?: 'schema-registry'
        
        def result = sh(
            script: """
                docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml exec -T ${schemaRegistryContainer} bash -c "
                    export KAFKA_OPTS=''
                    export JMX_PORT=''
                    export KAFKA_JMX_OPTS=''
                    unset JMX_PORT
                    unset KAFKA_JMX_OPTS
                    unset KAFKA_OPTS
                    kafka-topics --list --bootstrap-server ${kafkaServer} --command-config ${env.CLIENT_CONFIG_FILE} | grep -x '${params.TOPIC_NAME}'
                " 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        return result == params.TOPIC_NAME
    } catch (Exception e) {
        echo "⚠️ Could not verify topic existence: ${e.message}"
        return false
    }
}

def consumeAvroMessages() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    def schemaRegistryUrl = env.SCHEMA_REGISTRY_URL ?: params.SCHEMA_REGISTRY_URL
    def schemaRegistryContainer = params.SCHEMA_REGISTRY_CONTAINER ?: 'schema-registry'
    def offsetFlag = params.OFFSET_RESET == 'earliest' ? '--from-beginning' : ''
    
    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                     usernameVariable: 'KAFKA_USER', 
                                     passwordVariable: 'KAFKA_PASS')]) {
        
        def securityProps = buildSecurityProperties(env.KAFKA_USER, env.KAFKA_PASS)
        
        def result = sh(
            script: """
                docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml exec -T ${schemaRegistryContainer} bash -c '
                    # Completely disable JMX to avoid port conflicts
                    unset KAFKA_OPTS JMX_PORT KAFKA_JMX_OPTS KAFKA_HEAP_OPTS
                    export KAFKA_OPTS=""
                    export JMX_PORT=""
                    export KAFKA_JMX_OPTS=""
                    export KAFKA_HEAP_OPTS=""

                    # Consume Avro messages with timeout and filtering
                    timeout ${timeoutSeconds}s kafka-avro-console-consumer \\
                        --bootstrap-server ${kafkaServer} \\
                        --topic ${params.TOPIC_NAME} \\
                        ${offsetFlag} \\
                        --property schema.registry.url=${schemaRegistryUrl} \\
                        --property print.key=true \\
                        --property print.timestamp=true \\
                        --property key.separator=" | " \\
                        --consumer-property group.id=${params.CONSUMER_GROUP_ID} \\
                        --consumer-property auto.offset.reset=${params.OFFSET_RESET} \\
                        --consumer-property enable.auto.commit=true \\
                        --consumer-property auto.commit.interval.ms=1000 \\
                        --consumer-property session.timeout.ms=30000 \\
                        --consumer-property heartbeat.interval.ms=3000 \\
                        ${securityProps} \\
                        ${maxMsgFlag} \\
                        2>/dev/null | grep "^{" || echo "Consumer finished"
                '
            """,
            returnStdout: true
        )
        
        return result.trim()
    }
}

def buildSecurityProperties(username, password) {
    switch(params.SECURITY_PROTOCOL) {
        case 'SASL_PLAINTEXT':
        case 'SASL_SSL':
            return """--consumer-property security.protocol=${params.SECURITY_PROTOCOL} \\
                        --consumer-property sasl.mechanism=PLAIN \\
                        --consumer-property sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";'"""
        case 'PLAINTEXT':
            return "--consumer-property security.protocol=PLAINTEXT"
        default:
            return """--consumer-property security.protocol=SASL_PLAINTEXT \\
                        --consumer-property sasl.mechanism=PLAIN \\
                        --consumer-property sasl.jaas.config='org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";'"""
    }
}

def cleanupClientConfig() {
    try {
        def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
        def schemaRegistryContainer = params.SCHEMA_REGISTRY_CONTAINER ?: 'schema-registry'
        sh """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
            exec -T ${schemaRegistryContainer} bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}

def createKafkaClientConfig(username, password) {
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    def schemaRegistryContainer = params.SCHEMA_REGISTRY_CONTAINER ?: 'schema-registry'
    
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
            securityConfig = """
security.protocol=PLAINTEXT
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
                   !trimmed.startsWith('#') &&
                   (trimmed.startsWith('{') || trimmed.contains('|')) // JSON or has our key-value separator
        }
    
    def messageCount = messageLines.size()
    
    def content = """# Avro Kafka Consumer Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Messages Retrieved: ${messageCount}
# Duration: ${duration}ms
# Offset Reset: ${params.OFFSET_RESET}
# Schema Registry: ${env.SCHEMA_REGISTRY_URL}

"""

    if (messageCount == 0) {
        content += """No Avro messages found.

Possible reasons:
- Topic is empty
- Messages already consumed by this consumer group
- Offset setting (try 'earliest' to read from beginning)
- Consumer timeout reached
- Schema Registry connection issues
- Topic contains non-Avro messages

"""
    } else {
        content += """${'='*60}
AVRO MESSAGES
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
    }
    
    writeFile file: env.MESSAGES_FILE, text: content
    echo "Saved ${messageCount} Avro messages to ${env.MESSAGES_FILE}"
    
    // Create stats file
    def stats = [
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        messageCount: messageCount,
        duration: duration,
        timestamp: timestamp,
        schemaRegistry: env.SCHEMA_REGISTRY_URL,
        offsetReset: params.OFFSET_RESET
    ]
    
    writeFile file: env.STATS_FILE, text: groovy.json.JsonOutput.toJson(stats)
}