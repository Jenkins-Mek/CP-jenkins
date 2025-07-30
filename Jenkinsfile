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
        booleanParam(name: 'VERBOSE_OUTPUT', defaultValue: false, description: 'Include debug information in output')
    ])
])

pipeline {
    agent any
    
    environment {
        MESSAGES_FILE = 'consumed-avro-messages.txt'
        STATS_FILE = 'avro-consumption-stats.json'
        DEBUG_FILE = 'avro-consumer-debug.log'
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
                    echo "🔍 Verbose Output: ${params.VERBOSE_OUTPUT}"
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
                    def consumeResult = consumeAvroMessagesEnhanced()
                    def endTime = System.currentTimeMillis()
                    def duration = endTime - startTime
                    
                    saveAvroMessages(consumeResult.messages, consumeResult.debugInfo, duration)
                }
            }
        }
    }

    post {
        success {
            script {
                def artifacts = "${env.MESSAGES_FILE}, ${env.STATS_FILE}"
                if (params.VERBOSE_OUTPUT) {
                    artifacts += ", ${env.DEBUG_FILE}"
                }
                archiveArtifacts artifacts: artifacts, allowEmptyArchive: true
            }
            echo "✅ Avro message consumption completed!"
        }
        failure {
            echo "❌ Avro message consumption failed"
        }
        always {
            script {
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
                    curl -s --connect-timeout 10 --max-time 15 ${env.SCHEMA_REGISTRY_URL}/subjects || echo 'FAILED'
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
                    curl -s --connect-timeout 10 --max-time 15 ${env.SCHEMA_REGISTRY_URL}/subjects/${params.TOPIC_NAME}-value/versions/latest 2>/dev/null || echo 'NOT_FOUND'
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

def consumeAvroMessagesEnhanced() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    
    def securityProps = buildSecurityProperties()
    
    // Use either --from-beginning OR auto.offset.reset, not both
    def offsetFlag = ""
    def offsetProperty = ""
    if (params.OFFSET_RESET == 'earliest') {
        offsetFlag = "--from-beginning"
    } else {
        offsetProperty = "--consumer-property auto.offset.reset=${params.OFFSET_RESET}"
    }
    
    def result = sh(
        script: """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
            exec -T ${env.CONTAINER_NAME} bash -c '
                # Generate unique file names to avoid conflicts
                TIMESTAMP=\$(date +%s%N)
                MESSAGES_FILE="/tmp/kafka_messages_\${TIMESTAMP}"
                DEBUG_FILE="/tmp/kafka_debug_\${TIMESTAMP}"
                
                # Consume Avro messages with enhanced logging control
                (timeout ${timeoutSeconds}s kafka-avro-console-consumer \\
                    --bootstrap-server ${kafkaServer} \\
                    --topic ${params.TOPIC_NAME} \\
                    ${offsetFlag} \\
                    --property schema.registry.url=${env.SCHEMA_REGISTRY_URL} \\
                    --consumer-property group.id=${params.CONSUMER_GROUP_ID} \\
                    ${offsetProperty} \\
                    --consumer-property enable.auto.commit=true \\
                    --consumer-property auto.commit.interval.ms=1000 \\
                    --consumer-property session.timeout.ms=30000 \\
                    --consumer-property heartbeat.interval.ms=3000 \\
                    --consumer-property fetch.min.bytes=1 \\
                    --consumer-property fetch.max.wait.ms=5000 \\
                    ${securityProps} \\
                    ${maxMsgFlag} \\
                    --property print.key=true \\
                    --property print.timestamp=true \\
                    --property key.separator=" | " \\
                    --timeout-ms 10000 \\
                    2>\$DEBUG_FILE || echo "CONSUMER_FINISHED") | \\
                    grep -v -E "^[[:space:]]*\$" | \\
                    grep -v -E "(WARN|ERROR|INFO|DEBUG|TRACE)" | \\
                    grep -v -E "(SLF4J|org\\.|java\\.|kafka\\.|avro\\.)" | \\
                    grep -v -E "(Class path|binding|config|metrics)" | \\
                    grep -v -E "(NetworkClient|Login|Failed|Exception|Caused)" | \\
                    grep -v -E "\\[20[0-9][0-9]-[0-9][0-9]-[0-9][0-9]" | \\
                    grep -v -E "^#" | \\
                    grep -v -E "CONSUMER_FINISHED" > \$MESSAGES_FILE
                
                # Output results with delimiter
                echo "===MESSAGES_START==="
                cat \$MESSAGES_FILE 2>/dev/null || echo ""
                echo "===MESSAGES_END==="
                echo "===DEBUG_START==="
                if [ "${params.VERBOSE_OUTPUT}" = "true" ]; then
                    cat \$DEBUG_FILE 2>/dev/null || echo "No debug info available"
                else
                    echo "Debug output disabled (enable VERBOSE_OUTPUT to see)"
                fi
                echo "===DEBUG_END==="
                
                # Cleanup temp files
                rm -f \$MESSAGES_FILE \$DEBUG_FILE
            '
        """,
        returnStdout: true
    )
    
    // Parse the result to separate messages and debug info
    def lines = result.split('\n')
    def messages = []
    def debugInfo = []
    def currentSection = 'none'
    
    lines.each { line ->
        if (line == '===MESSAGES_START===') {
            currentSection = 'messages'
        } else if (line == '===MESSAGES_END===') {
            currentSection = 'none'
        } else if (line == '===DEBUG_START===') {
            currentSection = 'debug'
        } else if (line == '===DEBUG_END===') {
            currentSection = 'none'
        } else if (currentSection == 'messages' && line.trim()) {
            messages.add(line)
        } else if (currentSection == 'debug') {
            debugInfo.add(line)
        }
    }
    
    return [
        messages: messages.join('\n'),
        debugInfo: debugInfo.join('\n')
    ]
}

def buildSecurityProperties() {
    def securityProps = ""
    
    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                     usernameVariable: 'KAFKA_USER', 
                                     passwordVariable: 'KAFKA_PASS')]) {
        switch(params.SECURITY_PROTOCOL) {
            case 'SASL_PLAINTEXT':
            case 'SASL_SSL':
                // Use single quotes to prevent shell interpretation issues
                securityProps = """--consumer-property security.protocol=${params.SECURITY_PROTOCOL} \\
                    --consumer-property sasl.mechanism=PLAIN \\
                    --consumer-property 'sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";'"""
                break
            case 'PLAINTEXT':
                securityProps = "--consumer-property security.protocol=PLAINTEXT"
                break
            default:
                securityProps = """--consumer-property security.protocol=SASL_PLAINTEXT \\
                    --consumer-property sasl.mechanism=PLAIN \\
                    --consumer-property 'sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";'"""
                break
        }
    }
    
    return securityProps
}

def saveAvroMessages(messages, debugInfo, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Process messages - they should already be clean from the enhanced consumer
    def messageLines = messages ? messages.split('\n').findAll { it.trim() } : []
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
# Verbose Output: ${params.VERBOSE_OUTPUT}

"""

    if (messageCount == 0) {
        content += """No Avro messages found.

Possible reasons:
- Topic is empty or contains no Avro messages
- Messages already consumed by this consumer group (try different group ID)
- Offset setting ('earliest' reads from beginning, 'latest' reads new messages only)
- Consumer timeout reached before messages arrived
- Schema incompatibility or deserialization issues
- Topic doesn't use Avro serialization
- Network connectivity issues with Kafka or Schema Registry

To troubleshoot:
1. Enable VERBOSE_OUTPUT parameter to see debug information
2. Try with OFFSET_RESET='earliest' and a new CONSUMER_GROUP_ID
3. Verify topic has messages: kafka-topics --describe --topic ${params.TOPIC_NAME}
4. Check Schema Registry connectivity and topic schema registration

"""
    } else {
        content += """${'='*60}
AVRO MESSAGES
${'='*60}

"""
        messageLines.eachWithIndex { message, index ->
            def cleanMessage = message.trim()
            if (cleanMessage.contains(' | ')) {
                // Parse structured message with timestamp, key, and value
                def parts = cleanMessage.split(' \\| ', 3)
                if (parts.length >= 3) {
                    def timestamp_part = parts[0]
                    def key_part = parts[1] == 'null' ? '(no key)' : parts[1]
                    def value_part = parts[2]
                    
                    content += """[${index + 1}] Timestamp: ${timestamp_part}
Key: ${key_part}
Avro Value: ${value_part}

"""
                } else {
                    content += "[${index + 1}] ${cleanMessage}\n\n"
                }
            } else {
                // Message without separator, likely just the value
                content += """[${index + 1}] 
Avro Value: ${cleanMessage}

"""
            }
        }
    }
    
    // Generate comprehensive statistics
    def stats = [
        timestamp: timestamp,
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        schemaRegistry: env.SCHEMA_REGISTRY_URL,
        kafkaBootstrapServer: env.KAFKA_SERVER,
        messageCount: messageCount,
        durationMs: duration,
        offsetReset: params.OFFSET_RESET,
        securityProtocol: params.SECURITY_PROTOCOL,
        maxMessages: params.MAX_MESSAGES,
        timeoutSeconds: params.TIMEOUT_SECONDS,
        verboseOutput: params.VERBOSE_OUTPUT,
        composeDir: env.COMPOSE_DIR
    ]
    
    writeFile file: env.MESSAGES_FILE, text: content
    writeFile file: env.STATS_FILE, text: groovy.json.JsonBuilder(stats).toPrettyString()
    
    // Save debug information if verbose output is enabled
    if (params.VERBOSE_OUTPUT && debugInfo) {
        def debugContent = """# Kafka Avro Consumer Debug Log
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}

${debugInfo}
"""
        writeFile file: env.DEBUG_FILE, text: debugContent
        echo "🐛 Debug information saved to ${env.DEBUG_FILE}"
    }
    
    echo "📊 Saved ${messageCount} Avro messages to ${env.MESSAGES_FILE}"
    echo "📈 Statistics saved to ${env.STATS_FILE}"
    
    if (messageCount > 0) {
        echo "✅ Successfully consumed ${messageCount} Avro messages in ${duration}ms"
    } else {
        echo "⚠️ No messages consumed - check debug output or enable verbose mode for troubleshooting"
    }
}