properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-simple-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], description: 'Where to start consuming'),
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds')
    ])
])

pipeline {
    agent any
    
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        KAFKA_SERVER = 'localhost:9092'
        MESSAGES_FILE = 'consumed-messages.txt'
        STATS_FILE = 'consumption-stats.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    echo "✅ Topic: ${params.TOPIC_NAME}"
                    echo "📊 Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "⏰ Timeout: ${params.TIMEOUT_SECONDS}s"
                    echo "📝 Max Messages: ${params.MAX_MESSAGES}"
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
                        
                        saveMessages(messages, duration)
                        generateStats(messages, duration)
                    }
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: "${env.MESSAGES_FILE}, ${env.STATS_FILE}", allowEmptyArchive: true
            echo "✅ Message consumption completed!"
            script {
                def stats = readFile(env.STATS_FILE)
                echo "📊 Summary:\n${stats}"
            }
        }
        failure {
            echo "❌ Message consumption failed"
        }
        always {
            sh "rm -f /tmp/simple-consumer.properties || true"
        }
    }
}

def checkTopicExists() {
    try {
        def result = sh(
            script: """
                docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                    kafka-topics --bootstrap-server ${env.KAFKA_SERVER} --list | grep -x "${params.TOPIC_NAME}"
                '
            """,
            returnStdout: true
        ).trim()
        return result == params.TOPIC_NAME
    } catch (Exception e) {
        echo "⚠️ Could not verify topic existence: ${e.message}"
        return false
    }
}

def consumeMessages() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    
    def result = sh(
        script: """
            docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                # Clear JVM options
                unset KAFKA_OPTS JMX_PORT KAFKA_JMX_OPTS
                
                # Create simple consumer properties
                cat > /tmp/simple-consumer.properties << EOF
bootstrap.servers=${env.KAFKA_SERVER}
group.id=${params.CONSUMER_GROUP_ID}
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
auto.offset.reset=${params.OFFSET_RESET}
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${env.KAFKA_USER}" password="${env.KAFKA_PASS}";
enable.auto.commit=true
auto.commit.interval.ms=1000
session.timeout.ms=30000
heartbeat.interval.ms=3000
EOF

                # Consume messages
                timeout ${timeoutSeconds}s kafka-console-consumer \\
                    --bootstrap-server ${env.KAFKA_SERVER} \\
                    --topic "${params.TOPIC_NAME}" \\
                    --consumer.config /tmp/simple-consumer.properties \\
                    ${maxMsgFlag} \\
                    --property print.key=true \\
                    --property print.timestamp=true \\
                    --property key.separator=" | " \\
                    --timeout-ms 10000 2>/dev/null || echo "Consumer finished"
            '
        """,
        returnStdout: true
    )
    
    return result.trim()
}

def saveMessages(messages, duration) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    
    // Filter out system messages and keep only actual Kafka messages
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && 
                   !trimmed.contains('Consumer finished') &&
                   !trimmed.contains('WARN') &&
                   !trimmed.contains('ERROR') &&
                   !trimmed.startsWith('#') &&
                   trimmed.contains('|') // Has our key-value separator
        }
    
    def messageCount = messageLines.size()
    
    def content = """# Simple Kafka Consumer Report
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Consumer Group: ${params.CONSUMER_GROUP_ID}
# Messages Retrieved: ${messageCount}
# Duration: ${duration}ms
# Offset Reset: ${params.OFFSET_RESET}

"""

    if (messageCount == 0) {
        content += """⚠️ No messages found.

Possible reasons:
- Topic is empty
- Messages already consumed by this consumer group
- Offset setting (try 'earliest' to read from beginning)
- Consumer timeout reached

"""
    } else {
        content += """${'='*60}
MESSAGES
${'='*60}

"""
        messageLines.eachWithIndex { message, index ->
            def parts = message.split(' \\| ', 3)
            if (parts.length >= 3) {
                content += """[${index + 1}] ${parts[0]}
Key: ${parts[1] == 'null' ? '(no key)' : parts[1]}
Value: ${parts[2]}

"""
            } else {
                content += "[${index + 1}] ${message}\n\n"
            }
        }
    }
    
    writeFile file: env.MESSAGES_FILE, text: content
    echo "✅ Saved ${messageCount} messages to ${env.MESSAGES_FILE}"
}

def generateStats(messages, duration) {
    def messageLines = messages.split('\n')
        .findAll { line -> 
            def trimmed = line.trim()
            return trimmed && trimmed.contains('|') && 
                   !trimmed.contains('ERROR') && 
                   !trimmed.contains('Consumer finished')
        }
    
    def stats = [
        topic: params.TOPIC_NAME,
        consumerGroup: params.CONSUMER_GROUP_ID,
        messageCount: messageLines.size(),
        durationMs: duration,
        offsetReset: params.OFFSET_RESET,
        maxMessages: params.MAX_MESSAGES,
        timestamp: new Date().format('yyyy-MM-dd HH:mm:ss')
    ]
    
    def statsJson = groovy.json.JsonBuilder(stats).toPrettyString()
    writeFile file: env.STATS_FILE, text: statsJson
}