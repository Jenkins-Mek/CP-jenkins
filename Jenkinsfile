properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], defaultValue: 'latest', description: 'Where to start consuming'),
        choice(name: 'CONSUMER_MODE', choices: ['STRING', 'JSON_SCHEMA', 'AVRO_SCHEMA', 'BYTE_ARRAY'], defaultValue: 'STRING', description: 'Message format'),
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds')
    ])
])

pipeline {
    agent any
    
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        KAFKA_SERVER = 'localhost:9092'
        SCHEMA_REGISTRY = 'http://schema-registry:8081'
        MESSAGES_FILE = 'consumed-messages.txt'
        STATS_FILE = 'consumption-stats.json'
    }

    stages {
        stage('Validate') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    echo "✅ Consuming from topic: ${params.TOPIC_NAME}"
                    echo "📊 Consumer group: ${params.CONSUMER_GROUP_ID}"
                    echo "🔧 Mode: ${params.CONSUMER_MODE}"
                    echo "⏰ Timeout: ${params.TIMEOUT_SECONDS}s"
                }
            }
        }

        stage('Check Topic Exists') {
            steps {
                script {
                    def topicExists = checkTopicExists()
                    if (!topicExists) {
                        error("❌ Topic '${params.TOPIC_NAME}' does not exist")
                    }
                    echo "✅ Topic exists and is accessible"
                }
            }
        }

        stage('Consume Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', 
                                                   usernameVariable: 'KAFKA_USER', 
                                                   passwordVariable: 'KAFKA_PASS')]) {
                        def startTime = System.currentTimeMillis()
                        def messages = consumeMessages()
                        def endTime = System.currentTimeMillis()
                        def duration = endTime - startTime
                        
                        saveMessagesToFile(messages, duration)
                        generateStats(messages, duration)
                    }
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: "${env.MESSAGES_FILE}, ${env.STATS_FILE}", allowEmptyArchive: true
            echo "✅ Message consumption completed and archived!"
            script {
                def stats = readFile(env.STATS_FILE)
                echo "📊 Consumption Summary:\n${stats}"
            }
        }
        failure {
            echo "❌ Message consumption failed"
        }
        always {
            // Clean up temporary files
            sh "rm -f /tmp/consumer-*.properties || true"
        }
    }
}

def checkTopicExists() {
    try {
        def result = sh(
            script: """
                docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                    export KAFKA_OPTS=""
                    kafka-topics --bootstrap-server ${env.KAFKA_SERVER} --list | grep "^${params.TOPIC_NAME}\$"
                '
            """,
            returnStdout: true
        ).trim()
        return result == params.TOPIC_NAME
    } catch (Exception e) {
        return false
    }
}

def consumeMessages() {
    def deserializers = getDeserializers()
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    
    def result = sh(
        script: """
            docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                # Clear JVM options to prevent agent conflicts
                export KAFKA_OPTS=""
                export JMX_PORT=""
                export KAFKA_JMX_OPTS=""
                unset JMX_PORT
                unset KAFKA_JMX_OPTS
                unset KAFKA_OPTS
                
                cat > /tmp/consumer-jenkins.properties << EOF
bootstrap.servers=${env.KAFKA_SERVER}
group.id=${params.CONSUMER_GROUP_ID}
key.deserializer=${deserializers.key}
value.deserializer=${deserializers.value}
auto.offset.reset=${params.OFFSET_RESET}
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";
${getSchemaConfig()}
enable.auto.commit=true
auto.commit.interval.ms=1000
session.timeout.ms=30000
heartbeat.interval.ms=3000
EOF

                # Consume messages with proper error handling
                timeout ${timeoutSeconds}s kafka-console-consumer \\
                    --bootstrap-server ${env.KAFKA_SERVER} \\
                    --topic "${params.TOPIC_NAME}" \\
                    --consumer.config /tmp/consumer-jenkins.properties \\
                    ${maxMsgFlag} \\
                    --property print.key=true \\
                    --property print.timestamp=true \\
                    --property key.separator=" | " \\
                    --timeout-ms 10000 2>/dev/null || true
            '
        """,
        returnStdout: true
    )
    
    return result.trim()
}

def saveMessagesToFile(messages, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Enhanced message filtering
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('FATAL') && 
                   !trimmed.contains('ERROR') && 
                   !trimmed.contains('WARN') &&
                   !trimmed.contains('Native frames') &&
                   !trimmed.contains('libjvm') &&
                   !trimmed.contains('libjli') &&
                   !trimmed.contains('JavaMain') &&
                   !trimmed.contains('Aborted') &&
                   !trimmed.contains('JNI_') &&
                   !trimmed.contains('jni_') &&
                   !trimmed.contains('Threads::') &&
                   !trimmed.contains('JvmtiExport') &&
                   !trimmed.contains('Processed a total of') &&
                   !trimmed.startsWith('#') &&
                   trimmed.contains('|') // Ensure it has our separator
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
# Duration: ${duration}ms
# Timeout: ${params.TIMEOUT_SECONDS}s

"""

    if (messageCount == 0) {
        content += """⚠️ No messages consumed.
This could mean:
- Topic is empty
- All messages already consumed by this consumer group
- Messages are newer than the offset reset policy (try 'earliest')
- Consumer timeout reached before messages arrived
- Authentication/authorization issues

Troubleshooting suggestions:
1. Check if topic has messages: kafka-console-consumer --from-beginning
2. Try different consumer group ID
3. Verify topic permissions
4. Check broker connectivity

"""
    } else {
        content += """${'='*80}
CONSUMED MESSAGES
${'='*80}

"""
        messageLines.eachWithIndex { message, index ->
            def parts = message.split(' \\| ', 3)
            if (parts.length >= 3) {
                content += """Message ${index + 1}:
  Timestamp: ${parts[0]}
  Key: ${parts[1] == 'null' ? '(null)' : parts[1]}
  Value: ${parts[2]}

"""
            } else {
                content += "Message ${index + 1}:\n${message}\n\n"
            }
        }
        
        content += """${'='*80}
SUMMARY
${'='*80}
• Total messages consumed: ${messageCount}
• Topic: ${params.TOPIC_NAME}
• Consumer Group: ${params.CONSUMER_GROUP_ID}
• Consumer Mode: ${params.CONSUMER_MODE}
• Consumption time: ${timestamp}
• Duration: ${duration}ms
• Average time per message: ${messageCount > 0 ? Math.round(duration / messageCount) : 0}ms

"""
    }
    
    writeFile file: env.MESSAGES_FILE, text: content
    echo "✅ Messages saved to: ${env.MESSAGES_FILE} (${messageCount} messages)"
}

def generateStats(messages, duration) {
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && trimmed.contains('|') && !trimmed.contains('ERROR')
        }
    
    def stats = [
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        consumerMode: params.CONSUMER_MODE,
        messageCount: messageLines.size(),
        durationMs: duration,
        offsetReset: params.OFFSET_RESET,
        maxMessages: params.MAX_MESSAGES,
        timestamp: new Date().format('yyyy-MM-dd HH:mm:ss'),
        avgTimePerMessage: messageLines.size() > 0 ? Math.round(duration / messageLines.size()) : 0
    ]
    
    def statsJson = groovy.json.JsonBuilder(stats).toPrettyString()
    writeFile file: env.STATS_FILE, text: statsJson
}

def getDeserializers() {
    switch (params.CONSUMER_MODE) {
        case 'JSON_SCHEMA':
            return [
                key: 'org.apache.kafka.common.serialization.StringDeserializer',
                value: 'io.confluent.kafka.serializers.json.KafkaJsonSchemaDeserializer'
            ]
        case 'AVRO_SCHEMA':
            return [
                key: 'org.apache.kafka.common.serialization.StringDeserializer',
                value: 'io.confluent.kafka.serializers.KafkaAvroDeserializer'
            ]
        case 'BYTE_ARRAY':
            return [
                key: 'org.apache.kafka.common.serialization.ByteArrayDeserializer',
                value: 'org.apache.kafka.common.serialization.ByteArrayDeserializer'
            ]
        default: // STRING
            return [
                key: 'org.apache.kafka.common.serialization.StringDeserializer',
                value: 'org.apache.kafka.common.serialization.StringDeserializer'
            ]
    }
}

def getSchemaConfig() {
    if (params.CONSUMER_MODE in ['JSON_SCHEMA', 'AVRO_SCHEMA']) {
        return """schema.registry.url=${env.SCHEMA_REGISTRY}
auto.register.schemas=false
use.latest.version=true"""
    }
    return ""
}