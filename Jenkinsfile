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
        choice(name: 'OUTPUT_FORMAT', choices: ['json-only', 'all-messages', 'filtered'], description: 'Message output format')
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
                    if (!params.TOPIC_NAME?.trim()) {
                        error("TOPIC_NAME is required")
                    }
                    echo "Topic: ${params.TOPIC_NAME}"
                    echo "Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "Timeout: ${params.TIMEOUT_SECONDS}s"
                    echo "Max Messages: ${params.MAX_MESSAGES}"
                    echo "Kafka Server: ${params.KAFKA_BOOTSTRAP_SERVER}"
                    echo "Schema Registry: ${params.SCHEMA_REGISTRY_URL}"
                    echo "Output Format: ${params.OUTPUT_FORMAT}"
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
            echo "Avro message consumption completed successfully"
        }
        failure {
            echo "Avro message consumption failed"
        }
        always {
            echo "Pipeline execution completed"
        }
    }
}

def consumeAvroMessages() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    def composeDir = params.COMPOSE_DIR
    
    def offsetFlag = ""
    if (params.OFFSET_RESET == 'earliest') {
        offsetFlag = "--from-beginning"
    }
    
    def securityProps = buildSecurityProps()
    def outputFilter = buildOutputFilter()
    
    def result = sh(
        script: """
            cd ${composeDir}
            timeout ${timeoutSeconds}s docker exec -i ${env.CONTAINER_NAME} kafka-avro-console-consumer \\
                --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                --topic ${params.TOPIC_NAME} \\
                ${offsetFlag} \\
                --property schema.registry.url=${params.SCHEMA_REGISTRY_URL} \\
                --consumer-property group.id=${params.CONSUMER_GROUP_ID} \\
                ${securityProps} \\
                ${maxMsgFlag} \\
                ${outputFilter} || echo "CONSUMER_FINISHED"
        """,
        returnStdout: true
    )
    
    return result
}

def buildSecurityProps() {
    def securityProps = ""
    
    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                     usernameVariable: 'KAFKA_USER', 
                                     passwordVariable: 'KAFKA_PASS')]) {
        switch(params.SECURITY_PROTOCOL) {
            case 'SASL_PLAINTEXT':
            case 'SASL_SSL':
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

def buildOutputFilter() {
    switch(params.OUTPUT_FORMAT) {
        case 'json-only':
            return "2>/dev/null | grep '^{'"
        case 'all-messages':
            return "2>&1"
        case 'filtered':
            return "2>/dev/null | grep -E '^(\\{|\\[)'"
        default:
            return "2>/dev/null | grep '^{'"
    }
}

def saveAvroMessages(rawOutput, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Clean up the output based on format choice
    def messageLines = []
    if (params.OUTPUT_FORMAT == 'all-messages') {
        // Keep everything but filter out known noise
        messageLines = rawOutput.split('\n')
            .findAll { line -> 
                def trimmed = line.trim()
                return trimmed && 
                       !trimmed.contains('CONSUMER_FINISHED') &&
                       !trimmed.startsWith('SLF4J:') &&
                       !trimmed.contains('Class path contains multiple') &&
                       !trimmed.contains('log4j:WARN') &&
                       trimmed.length() > 0
            }
    } else {
        // For json-only and filtered, the output should already be clean
        messageLines = rawOutput.split('\n')
            .findAll { line -> 
                def trimmed = line.trim()
                return trimmed && 
                       !trimmed.contains('CONSUMER_FINISHED') &&
                       trimmed.length() > 0
            }
    }
    
    def messageCount = messageLines.size()
    
    def content = """# Kafka Avro Consumer Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Schema Registry: ${params.SCHEMA_REGISTRY_URL}
# Messages Retrieved: ${messageCount}
# Duration: ${duration}ms
# Offset Reset: ${params.OFFSET_RESET}
# Security Protocol: ${params.SECURITY_PROTOCOL}
# Output Format: ${params.OUTPUT_FORMAT}

"""

    if (messageCount == 0) {
        content += """No messages found.

Possible reasons:
- Topic is empty
- Messages already consumed by this consumer group
- Using 'latest' offset and no new messages
- Consumer timeout reached
- Messages don't match the output filter

Try using 'earliest' offset, different consumer group ID, or 'all-messages' output format.

"""
    } else {
        content += """${'='*60}
CONSUMED MESSAGES
${'='*60}

"""
        messageLines.eachWithIndex { message, index ->
            content += "[${index + 1}] ${message}\n"
        }
    }
    
    def stats = [
        timestamp: timestamp,
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        schemaRegistry: params.SCHEMA_REGISTRY_URL,
        messageCount: messageCount,
        durationMs: duration,
        offsetReset: params.OFFSET_RESET,
        securityProtocol: params.SECURITY_PROTOCOL,
        outputFormat: params.OUTPUT_FORMAT
    ]
    
    writeFile file: env.MESSAGES_FILE, text: content
    writeFile file: env.STATS_FILE, text: groovy.json.JsonBuilder(stats).toPrettyString()
    
    echo "Saved ${messageCount} messages to ${env.MESSAGES_FILE}"
    echo "Statistics saved to ${env.STATS_FILE}"
}