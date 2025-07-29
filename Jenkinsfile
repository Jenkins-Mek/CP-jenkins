properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-simple-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], description: 'Where to start consuming'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL', 'PLAINTEXT'], description: 'Security protocol'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker compose directory'),
        string(name: 'TIMEOUT_SECONDS', defaultValue: '30', description: 'Consumer timeout in seconds')
    ])
])

pipeline {
    agent any
    
    environment {
        CLIENT_CONFIG_FILE = '/tmp/simple-consumer.properties'
        MESSAGES_FILE = 'consumed-messages.txt'
        STATS_FILE = 'consumption-stats.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    // Set environment variables with defaults
                    env.COMPOSE_DIR = params.COMPOSE_DIR ?: '/confluent/cp-mysetup/cp-all-in-one'
                    env.KAFKA_SERVER = params.KAFKA_BOOTSTRAP_SERVER ?: 'localhost:9092'
                    
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME is required")
                    }
                    echo "✅ Topic: ${params.TOPIC_NAME}"
                    echo "📊 Consumer Group: ${params.CONSUMER_GROUP_ID}"
                    echo "⏰ Timeout: ${params.TIMEOUT_SECONDS}s"
                    echo "📝 Max Messages: ${params.MAX_MESSAGES}"
                    echo "🏠 Compose Dir: ${env.COMPOSE_DIR}"
                    echo "🌐 Kafka Server: ${env.KAFKA_SERVER}"
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

        stage('Consume Messages') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()
                    def messages = consumeMessages()
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
        
        def result = sh(
            script: """
                docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml exec -T broker bash -c "
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

def consumeMessages() {
    def maxMsgs = params.MAX_MESSAGES.toInteger()
    def maxMsgFlag = maxMsgs > 0 ? "--max-messages ${maxMsgs}" : ""
    def timeoutSeconds = params.TIMEOUT_SECONDS.toInteger()
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    
    def result = sh(
        script: """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml exec -T broker bash -c '
                # Completely disable JMX to avoid port conflicts
                unset KAFKA_OPTS JMX_PORT KAFKA_JMX_OPTS KAFKA_HEAP_OPTS
                export KAFKA_OPTS=""
                export JMX_PORT=""
                export KAFKA_JMX_OPTS=""
                export KAFKA_HEAP_OPTS=""
                
                # Add consumer-specific settings to existing client config
                cat >> ${env.CLIENT_CONFIG_FILE} << EOF
group.id=${params.CONSUMER_GROUP_ID}
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
auto.offset.reset=${params.OFFSET_RESET}
enable.auto.commit=true
auto.commit.interval.ms=1000
session.timeout.ms=30000
heartbeat.interval.ms=3000
# Disable JMX to avoid port conflicts
jmx.port=
EOF

                # Consume messages with JMX disabled
                timeout ${timeoutSeconds}s kafka-console-consumer \\
                    --bootstrap-server ${kafkaServer} \\
                    --topic ${params.TOPIC_NAME} \\
                    --consumer.config ${env.CLIENT_CONFIG_FILE} \\
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

def cleanupClientConfig() {
    try {
        def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
        sh """
            docker compose --project-directory ${composeDir} -f ${composeDir}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}

def createKafkaClientConfig(username, password) {
    def composeDir = env.COMPOSE_DIR ?: params.COMPOSE_DIR
    def kafkaServer = env.KAFKA_SERVER ?: params.KAFKA_BOOTSTRAP_SERVER
    
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
        exec -T broker bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${kafkaServer}
${securityConfig}
EOF'
    """
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
