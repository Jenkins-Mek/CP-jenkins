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
                2>/dev/null | grep '^{' || echo "CONSUMER_FINISHED"
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

def saveAvroMessages(rawOutput, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Clean JSON messages only
    def messageLines = rawOutput.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('CONSUMER_FINISHED') &&
                   trimmed.startsWith('{') &&
                   trimmed.length() > 2
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

"""

    if (messageCount == 0) {
        content += """No JSON messages found.

Possible reasons:
- Topic is empty
- Messages already consumed by this consumer group
- Using 'latest' offset and no new messages
- Consumer timeout reached
- Messages are not in JSON format

Try using 'earliest' offset or different consumer group ID.

"""
    } else {
        content += """${'='*60}
JSON MESSAGES
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
        securityProtocol: params.SECURITY_PROTOCOL
    ]
    
    // Fix the JsonBuilder instantiation
    def jsonBuilder = new groovy.json.JsonBuilder(stats)
    
    writeFile file: env.MESSAGES_FILE, text: content
    writeFile file: env.STATS_FILE, text: jsonBuilder.toPrettyString()
    
    echo "Saved ${messageCount} messages to ${env.MESSAGES_FILE}"
    echo "Statistics saved to ${env.STATS_FILE}"
}