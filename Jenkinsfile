properties([
    parameters([
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name (required)'),
        string(name: 'CONSUMER_GROUP_ID', defaultValue: 'jenkins-consumer', description: 'Consumer group ID'),
        string(name: 'MAX_MESSAGES', defaultValue: '100', description: 'Max messages to consume (0 = unlimited)'),
        choice(name: 'OFFSET_RESET', choices: ['latest', 'earliest'], defaultValue: 'latest', description: 'Where to start consuming'),
        choice(name: 'CONSUMER_MODE', choices: ['STRING', 'JSON_SCHEMA', 'AVRO_SCHEMA'], defaultValue: 'STRING', description: 'Message format'),
        booleanParam(name: 'SAVE_TO_FILE', defaultValue: true, description: 'Save messages to file')
    ])
])

pipeline {
    agent any
    
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        KAFKA_SERVER = 'localhost:9092'
        SCHEMA_REGISTRY = 'http://schema-registry:8081'
        OUTPUT_FILE = 'consumed-messages.json'
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
                        consumeMessages()
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.SAVE_TO_FILE) {
                    archiveArtifacts artifacts: "${env.OUTPUT_FILE}", allowEmptyArchive: true
                }
                echo "✅ Message consumption completed!"
            }
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
    
    try {
        def result = sh(
            script: """
                docker compose -f ${env.COMPOSE_DIR}/docker-compose.yml exec -T broker bash -c '
                    # Create consumer config
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

                    echo "🔽 Starting consumption from ${params.TOPIC_NAME}..."
                    
                    # Consume messages
                    timeout 60s kafka-console-consumer \\
                        --bootstrap-server ${env.KAFKA_SERVER} \\
                        --topic "${params.TOPIC_NAME}" \\
                        --consumer.config /tmp/consumer.properties \\
                        ${maxMsgFlag} \\
                        --property print.key=true \\
                        --property print.timestamp=true \\
                        --property key.separator=" | " > ${env.OUTPUT_FILE} 2>/dev/null || true
                    
                    # Count messages
                    MSG_COUNT=\$(wc -l < ${env.OUTPUT_FILE} 2>/dev/null || echo "0")
                    echo "📊 Consumed \$MSG_COUNT messages"
                    
                    # Show sample
                    if [ \$MSG_COUNT -gt 0 ]; then
                        echo "📝 Sample messages:"
                        head -3 ${env.OUTPUT_FILE}
                    fi
                '
            """,
            returnStdout: true
        )
        
        echo result
        
    } catch (Exception e) {
        error("Failed to consume messages: ${e.getMessage()}")
    }
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