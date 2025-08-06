import groovy.json.JsonOutput

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'broker:29093', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name'),
        string(name: 'SCHEMA_SUBJECT', defaultValue: '', description: 'Schema subject name (defaults to TOPIC_NAME-value if empty)'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://schema-registry:8081', description: 'Schema Registry URL'),
        text(name: 'MESSAGE_DATA', defaultValue: '{"message": "Hello World", "timestamp": "2024-01-01T00:00:00Z"}', description: 'Message data (must conform to existing schema)'),
        string(name: 'MESSAGE_COUNT', defaultValue: '1', description: 'Number of messages to produce'),
        booleanParam(name: 'VALIDATE_MESSAGE_FORMAT', defaultValue: true, description: 'Validate message JSON format before producing')
    ])
])

pipeline {
    agent any

    environment {
        PRODUCER_OUTPUT_FILE = 'schema-producer-results.txt'
        SCHEMA_ID = ''
        SCHEMA_TYPE = ''
        FINAL_SCHEMA_SUBJECT = ''
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("TOPIC_NAME parameter is required to produce messages.")
                    }

                    if (params.MESSAGE_COUNT && !params.MESSAGE_COUNT.isNumber()) {
                        error("MESSAGE_COUNT must be a valid number")
                    }

                    def messageCount = params.MESSAGE_COUNT.toInteger()
                    if (messageCount <= 0 || messageCount > 10000) {
                        error("MESSAGE_COUNT must be between 1 and 10000")
                    }

                    // Set schema subject - use parameter if provided, otherwise default to topic-value
                    // Set schema subject with robust null/empty checking
                    def schemaSubject = params.SCHEMA_SUBJECT ?: ""
                    def topicName = params.TOPIC_NAME ?: ""
                    
                    echo "DEBUG: After elvis - topicName = '${topicName}'"
                    echo "DEBUG: After elvis - schemaSubject = '${schemaSubject}'"
                    
                    def finalSchemaSubject = ""
                    if (schemaSubject.toString().trim() != "") {
                        finalSchemaSubject = schemaSubject.toString().trim()
                        echo "Using provided schema subject: ${finalSchemaSubject}"
                    } else {
                        finalSchemaSubject = "${topicName}-value"
                        echo "DEBUG: Calculated subject = ${finalSchemaSubject}"
                        echo "Using default schema subject: ${finalSchemaSubject}"
                    }
                    
                    // Set environment variable with simple assignment
                    env.FINAL_SCHEMA_SUBJECT = '${finalSchemaSubject}'

                    echo "Parameters validated successfully"
                    echo "Topic: ${params.TOPIC_NAME}"
                    echo "Schema Subject: ${env.FINAL_SCHEMA_SUBJECT}"
                    echo "Schema Registry: ${params.SCHEMA_REGISTRY_URL}"
                    echo "Message Count: ${messageCount}"
                }
            }
        }

        stage('Validate Schema Registry') {
            steps {
                script {
                    echo "Validating Schema Registry connection..."
                    validateSchemaRegistry()
                }
            }
        }

        stage('Retrieve Schema Information') {
            steps {
                script {
                    echo "Retrieving schema information for subject: ${env.FINAL_SCHEMA_SUBJECT}"
                    retrieveSchemaInfo()
                }
            }
        }

        stage('Prepare Message Data') {
            steps {
                script {
                    echo "Preparing message data..."
                    echo "Schema ID: ${env.SCHEMA_ID}"
                    echo "Schema Type: ${env.SCHEMA_TYPE}"
                    echo "Message Count: ${params.MESSAGE_COUNT}"
                    echo "Sample Message: ${params.MESSAGE_DATA}"
                    
                    if (params.VALIDATE_MESSAGE_FORMAT) {
                        validateMessageFormat()
                    }
                }
            }
        }

        stage('Produce Schema-Based Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        def topicName = params.TOPIC_NAME.trim()
                        echo "Producing schema-based messages to topic: ${topicName}"
                        def result = produceSchemaMessages(topicName, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                        echo result
                        saveProducerResults(result)
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                archiveArtifacts artifacts: "${env.PRODUCER_OUTPUT_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "Schema producer results archived successfully."
                echo "Schema-based message production completed successfully!"
            }
        }
        failure {
            script {
                echo "Schema-based message production failed. Check the logs above for details."
            }
        }
    }
}

def validateSchemaRegistry() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" ${params.SCHEMA_REGISTRY_URL}/subjects 2>/dev/null)
                if [ "\$RESPONSE" = "200" ]; then
                    echo "Schema Registry is accessible at ${params.SCHEMA_REGISTRY_URL}"
                else
                    echo "Schema Registry is not accessible (HTTP \$RESPONSE)"
                    exit 1
                fi
            '
        """
    } catch (Exception e) {
        error("Schema Registry validation failed: ${e.getMessage()}")
    }
}

def retrieveSchemaInfo() {
    try {
        def schemaInfo = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    echo "Retrieving schema information for subject: ${env.FINAL_SCHEMA_SUBJECT}"
                    RESPONSE=\$(curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${env.FINAL_SCHEMA_SUBJECT}/versions/latest)

                    if echo "\$RESPONSE" | grep -q "schema"; then
                        SCHEMA_ID=\$(echo "\$RESPONSE" | grep -o \'"id":[0-9]*\' | cut -d: -f2)
                        SCHEMA_TYPE=\$(echo "\$RESPONSE" | grep -o \'"schemaType":"[^"]*"\' | cut -d: -f2 | tr -d \'"\'')
                        
                        # Default to AVRO if no schemaType specified (Avro default)
                        if [ -z "\$SCHEMA_TYPE" ]; then
                            SCHEMA_TYPE="AVRO"
                        fi
                        
                        echo "SCHEMA_ID=\$SCHEMA_ID"
                        echo "SCHEMA_TYPE=\$SCHEMA_TYPE"
                        echo "Schema found successfully"
                    else
                        echo "ERROR: Schema not found for subject: ${env.FINAL_SCHEMA_SUBJECT}"
                        echo "Response: \$RESPONSE"
                        exit 1
                    fi
                '
            """,
            returnStdout: true
        ).trim()

        // Parse the output to extract schema ID and type
        def lines = schemaInfo.split('\n')
        lines.each { line ->
            if (line.startsWith('SCHEMA_ID=')) {
                env.SCHEMA_ID = line.split('=')[1]
            } else if (line.startsWith('SCHEMA_TYPE=')) {
                env.SCHEMA_TYPE = line.split('=')[1]
            }
        }

        if (!env.SCHEMA_ID) {
            error("Failed to retrieve schema ID for subject: ${env.FINAL_SCHEMA_SUBJECT}")
        }

        echo "Retrieved Schema ID: ${env.SCHEMA_ID}"
        echo "Retrieved Schema Type: ${env.SCHEMA_TYPE}"

    } catch (Exception e) {
        error("Failed to retrieve schema information: ${e.getMessage()}")
    }
}

def validateMessageFormat() {
    echo "Validating message data format..."
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T schema-registry bash -c '
            echo "Performing basic JSON format validation..."
            if command -v jq >/dev/null 2>&1; then
                echo '"${params.MESSAGE_DATA}"' | jq . >/dev/null
                if [ \$? -eq 0 ]; then
                    echo "Message data appears to be valid JSON format"
                else
                    echo "Message data is not valid JSON format"
                    exit 1
                fi
            else
                echo "jq not available - skipping JSON validation"
            fi
        '
    """
}

def produceSchemaMessages(topicName, username, password) {
    try {
        def producerCommand = getSchemaProducerCommand(env.SCHEMA_TYPE)
        def saslConfig = "org.apache.kafka.common.security.plain.PlainLoginModule required username=\\\"${username}\\\" password=\\\"${password}\\\";"
        
        // Generate message data based on count
        def messageCount = params.MESSAGE_COUNT.toInteger()
        def messageData = params.MESSAGE_DATA.trim()
        def messages = []
        for (int i = 0; i < messageCount; i++) {
            messages.add(messageData)
        }
        def messageContent = messages.join('\n')
        
        def produceOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T schema-registry ${producerCommand} \\
                --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                --topic "${topicName}" \\
                --property schema.registry.url=${params.SCHEMA_REGISTRY_URL} \\
                --producer-property security.protocol=${params.SECURITY_PROTOCOL} \\
                --producer-property sasl.mechanism=PLAIN \\
                --producer-property "sasl.jaas.config=${saslConfig}" \\
                --property "value.schema.id=${env.SCHEMA_ID}" <<'EOF'
${messageContent}
EOF
            """,
            returnStdout: true
        ).trim()

        return """Schema-based message production completed.
Topic: ${topicName}
Schema Type: ${env.SCHEMA_TYPE}
Schema Subject: ${env.FINAL_SCHEMA_SUBJECT}
Schema ID: ${env.SCHEMA_ID}
Message Count: ${messageCount}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}

Producer Output:
${produceOutput}"""

    } catch (Exception e) {
        return "ERROR: Failed to produce schema-based messages to topic '${topicName}' - ${e.getMessage()}"
    }
}

def getSchemaProducerCommand(schemaType) {
    switch (schemaType.toUpperCase()) {
        case 'JSON':
        case 'JSON_SCHEMA':
            return 'kafka-json-schema-console-producer'
        case 'AVRO':
        case 'AVRO_SCHEMA':
            return 'kafka-avro-console-producer'
        case 'PROTOBUF':
        case 'PROTOBUF_SCHEMA':
            return 'kafka-protobuf-console-producer'
        default:
            return 'kafka-avro-console-producer' // Default to Avro if unknown
    }
}

def saveProducerResults(result) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Kafka Schema-Based Producer Results
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Schema Type: ${env.SCHEMA_TYPE}
# Schema Subject: ${env.FINAL_SCHEMA_SUBJECT}
# Schema ID: ${env.SCHEMA_ID}
# Message Count: ${params.MESSAGE_COUNT}
# Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security Protocol: ${params.SECURITY_PROTOCOL}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

================================================================================
PRODUCER EXECUTION RESULTS
================================================================================

${result}

================================================================================
SCHEMA INFORMATION
================================================================================

Schema Type: ${env.SCHEMA_TYPE}
Schema Subject: ${env.FINAL_SCHEMA_SUBJECT}
Schema ID: ${env.SCHEMA_ID}
Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

================================================================================
CONFIGURATION SUMMARY
================================================================================

Topic Name: ${params.TOPIC_NAME}
Schema Subject: ${env.FINAL_SCHEMA_SUBJECT}
Message Count: ${params.MESSAGE_COUNT}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}

Sample Message Data:
${params.MESSAGE_DATA}

================================================================================
"""

    writeFile file: env.PRODUCER_OUTPUT_FILE, text: content
}