properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], defaultValue: 'latest', description: 'Where to start consuming'),
        choice(name: 'CONSUMER_MODE', choices: ['STRING', 'JSON_SCHEMA', 'AVRO_SCHEMA'], defaultValue: 'STRING', description: 'Message format')
    ])
])

pipeline {
    agent any
    
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        KAFKA_SERVER = 'localhost:9092'
        SCHEMA_REGISTRY = 'http://schema-registry:8081'
        MESSAGES_FILE = 'consumed-messages.txt'
    }

    stages {
        stage('Validate') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    echo "✅ Consuming from topic: ${params.TOPIC_NAME}"
                }
            }
        }

        stage('Consume Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                                   usernameVariable: 'KAFKA_USER', 
                                                   passwordVariable: 'KAFKA_PASS')]) {
                        def messages = consumeMessages()
                        saveMessagesToFile(messages)
                    }
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: "${env.MESSAGES_FILE}", allowEmptyArchive: true
            echo "✅ Message consumption completed and archived!"
        }
        failure {
            echo "❌ Message consumption failed"
        }
    }
}

def consumeMessages() {
    def deserializer = getDeserializer()
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    
    def result = sh(
        script: """
            docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                cat > /tmp/consumer.properties << EOF
bootstrap.servers=${env.KAFKA_SERVER}
group.id=${params.CONSUMER_GROUP_ID}
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=${deserializer}
auto.offset.reset=${params.OFFSET_RESET}
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";
${getSchemaConfig()}
EOF

                # Consume messages and filter out error messages
                timeout 30s kafka-console-consumer \\
                    --bootstrap-server ${env.KAFKA_SERVER} \\
                    --topic "${params.TOPIC_NAME}" \\
                    --consumer.config /tmp/consumer.properties \\
                    ${maxMsgFlag} \\
                    --property print.key=true \\
                    --property print.timestamp=true \\
                    --property key.separator=" | " \\
                    --timeout-ms 10000 2>/dev/null | \\
                    grep -v "FATAL\\|ERROR\\|Native frames\\|libjvm\\|libjli\\|JavaMain\\|JNI_\\|jni_\\|Aborted\\|Threads::\\|JvmtiExport" || true
            '
        """,
        returnStdout: true
    )
    
    return result.trim()
}

def saveMessagesToFile(messages) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Filter out any remaining error messages and empty lines
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('FATAL') && 
                   !trimmed.contains('ERROR') && 
                   !trimmed.contains('Native frames') &&
                   !trimmed.contains('libjvm') &&
                   !trimmed.contains('libjli') &&
                   !trimmed.contains('JavaMain') &&
                   !trimmed.contains('Aborted') &&
                   !trimmed.contains('JNI_') &&
                   !trimmed.contains('jni_') &&
                   !trimmed.contains('Threads::') &&
                   !trimmed.contains('JvmtiExport')
        }
    
    def messageCount = messageLines.size()
    
    def content = """# Kafka Messages Consumption Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Total messages: ${messageCount}
# Bootstrap server: ${env.KAFKA_SERVER}
# Offset reset: ${params.OFFSET_RESET}
# Max messages: ${params.MAX_MESSAGES}
# Consumer mode: ${params.CONSUMER_MODE}

"""

    if (messageCount == 0) {
        content += """⚠️ No messages consumed.
This could mean:
- Topic is empty
- All messages already consumed by this consumer group
- Messages are newer than the offset reset policy
- JVM errors prevented message consumption

"""
    } else {
        content += """${'='*80}
CONSUMED MESSAGES
${'='*80}

"""
        messageLines.eachWithIndex { message, index ->
            content += "Message ${index + 1}:\n${message}\n\n"
        }
        
        content += """${'='*80}
SUMMARY
${'='*80}
• Total messages consumed: ${messageCount}
• Topic: ${params.TOPIC_NAME}
• Consumer Group: ${params.CONSUMER_GROUP_ID}
• Consumption time: ${timestamp}

"""
    }
    
    writeFile file: env.MESSAGES_FILE, text: content
    echo "✅ Messages saved to: ${env.MESSAGES_FILE} (${messageCount} messages)"
}

def getDeserializer() {
    switch (params.CONSUMER_MODE) {
        case 'JSON_SCHEMA':
            return 'io.confluent.kafka.serializers.json.KafkaJsonSchemaDeserializer'
        case 'AVRO_SCHEMA':
            return 'io.confluent.kafka.serializers.KafkaAvroDeserializer'
        default:
            return 'org.apache.kafka.common.serialization.StringDeserializer'
    }
}

def getSchemaConfig() {
    if (params.CONSUMER_MODE != 'STRING') {
        return "schema.registry.url=${env.SCHEMA_REGISTRY}"
    }
    return ""
}