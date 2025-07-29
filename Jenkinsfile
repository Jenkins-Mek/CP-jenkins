import groovy.json.JsonOutput

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Kafka topic name to produce messages to'),
        choice(name: 'SCHEMA_TYPE', choices: ['JSON_SCHEMA', 'AVRO_SCHEMA', 'PROTOBUF_SCHEMA'], description: 'Schema type for message serialization'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://schema-registry:8081', description: 'Schema Registry URL'),
        string(name: 'SCHEMA_SUBJECT', defaultValue: '', description: 'Schema subject name (optional - defaults to topic-value)'),
        text(name: 'SCHEMA_DEFINITION', defaultValue: '', description: 'Schema definition (JSON Schema/Avro/Protobuf format)'),
        booleanParam(name: 'AUTO_REGISTER_SCHEMA', defaultValue: true, description: 'Automatically register schema if not exists'),
        text(name: 'MESSAGE_DATA', defaultValue: '{"message": "Hello World", "timestamp": "2024-01-01T00:00:00Z"}', description: 'Message data (must conform to schema)'),
        string(name: 'MESSAGE_COUNT', defaultValue: '1', description: 'Number of messages to produce'),
        booleanParam(name: 'USE_FILE_INPUT', defaultValue: false, description: 'Use file input instead of parameter data'),
        string(name: 'INPUT_FILE_PATH', defaultValue: '/tmp/input-messages.json', description: 'Path to input file (only used when USE_FILE_INPUT is true)'),
        booleanParam(name: 'VALIDATE_SCHEMA_COMPATIBILITY', defaultValue: true, description: 'Validate schema compatibility before producing')
    ])
])

pipeline {
    agent any

    environment {
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
        PRODUCER_OUTPUT_FILE = 'schema-producer-results.txt'
        MESSAGE_DATA_FILE = '/tmp/producer-messages.json'
        SCHEMA_FILE = '/tmp/schema-definition.json'
    }

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("❌ TOPIC_NAME parameter is required to produce messages.")
                    }
                    
                    if (!params.SCHEMA_DEFINITION?.trim()) {
                        error("❌ SCHEMA_DEFINITION parameter is required for schema-based production.")
                    }
                    
                    if (params.MESSAGE_COUNT && !params.MESSAGE_COUNT.isNumber()) {
                        error("❌ MESSAGE_COUNT must be a valid number")
                    }
                    
                    def messageCount = params.MESSAGE_COUNT.toInteger()
                    if (messageCount <= 0 || messageCount > 10000) {
                        error("❌ MESSAGE_COUNT must be between 1 and 10000")
                    }
                    
                    echo "✅ Parameters validated successfully"
                    echo "   Topic: ${params.TOPIC_NAME}"
                    echo "   Schema Type: ${params.SCHEMA_TYPE}"
                    echo "   Schema Registry: ${params.SCHEMA_REGISTRY_URL}"
                    echo "   Message Count: ${messageCount}"
                    echo "   Auto Register Schema: ${params.AUTO_REGISTER_SCHEMA}"
                }
            }
        }

        stage('Create Kafka Client Config') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Validate Schema Registry') {
            steps {
                script {
                    echo "🔍 Validating Schema Registry connection..."
                    validateSchemaRegistry()
                }
            }
        }

        stage('Prepare Schema') {
            steps {
                script {
                    echo "📋 Preparing schema definition..."
                    prepareSchemaDefinition()
                }
            }
        }

        stage('Register/Validate Schema') {
            steps {
                script {
                    def schemaSubject = getSchemaSubject()
                    echo "📝 Processing schema for subject: ${schemaSubject}"
                    
                    if (params.AUTO_REGISTER_SCHEMA) {
                        registerSchema(schemaSubject)
                    } else {
                        validateExistingSchema(schemaSubject)
                    }
                }
            }
        }

        stage('Prepare Message Data') {
            steps {
                script {
                    if (params.USE_FILE_INPUT) {
                        echo "📁 Using file input: ${params.INPUT_FILE_PATH}"
                        prepareMessageDataFromFile()
                    } else {
                        echo "📝 Using parameter input data"
                        prepareMessageDataFromParameter()
                    }
                    
                    if (params.VALIDATE_SCHEMA_COMPATIBILITY) {
                        validateMessageAgainstSchema()
                    }
                }
            }
        }

        stage('Produce Schema-Based Messages') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        def topicName = params.TOPIC_NAME.trim()
                        echo "🚀 Producing schema-based messages to topic: ${topicName}"
                        def result = produceSchemaMessages(topicName, env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                        echo result
                        saveProducerResults(result)
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupFiles()
            }
        }
        success {
            script {
                archiveArtifacts artifacts: "${env.PRODUCER_OUTPUT_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Schema producer results archived successfully."
                echo "✅ Schema-based message production completed successfully!"
            }
        }
        failure {
            script {
                echo "❌ Schema-based message production failed. Check the logs above for details."
            }
        }
    }
}

def produceSchemaMessages(topicName, username, password) {
    try {
        def valueSerializer = getSchemaSerializer(params.SCHEMA_TYPE)
        
        def produceOutput = sh(
            script: """
                docker compose --project-directory '${params.COMPOSE_DIR}' -f '${params.COMPOSE_DIR}/docker-compose.yml' \\
                exec -T broker bash -c '
                    set -e
                    unset JMX_PORT KAFKA_JMX_OPTS KAFKA_OPTS
                    
                    # Create producer configuration
                    cat > /tmp/producer.properties << "PRODUCER_EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=${valueSerializer}
security.protocol=${params.SECURITY_PROTOCOL}
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
schema.registry.url=${params.SCHEMA_REGISTRY_URL}
auto.register.schemas=${params.AUTO_REGISTER_SCHEMA}
use.latest.version=true
PRODUCER_EOF

                    echo "Producer configuration:"
                    cat /tmp/producer.properties
                    echo ""

                    echo "Schema type: ${params.SCHEMA_TYPE}"
                    echo "Schema subject: ${getSchemaSubject()}"
                    echo ""

                    echo "Message data preview:"
                    head -3 ${env.MESSAGE_DATA_FILE}
                    echo ""

                    MESSAGE_COUNT=\$(wc -l < ${env.MESSAGE_DATA_FILE})
                    echo "Producing \$MESSAGE_COUNT schema-based messages to topic ${topicName}..."

                    START_TIME=\$(date +%s)
                    kafka-console-producer --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} \\
                        --topic "${topicName}" \\
                        --producer.config /tmp/producer.properties < ${env.MESSAGE_DATA_FILE}
                    END_TIME=\$(date +%s)

                    DURATION=\$((END_TIME - START_TIME))
                    echo ""
                    echo "✅ Successfully produced \$MESSAGE_COUNT schema-based messages in \$DURATION seconds"
                    echo "   Topic: ${topicName}"
                    echo "   Schema Type: ${params.SCHEMA_TYPE}"
                    echo "   Serializer: ${valueSerializer}"
                    echo "   Schema Subject: ${getSchemaSubject()}"
                '
            """,
            returnStdout: true
        ).trim()

        return "✅ Schema-based messages produced successfully.\n${produceOutput}"

    } catch (Exception e) {
        return "ERROR: Failed to produce schema-based messages to topic '${topicName}' - ${e.getMessage()}"
    }
}

def getSchemaSerializer(schemaType) {
    switch (schemaType.toLowerCase()) {
        case 'json_schema':
            return 'io.confluent.kafka.serializers.json.KafkaJsonSchemaSerializer'
        case 'avro_schema':
            return 'io.confluent.kafka.serializers.KafkaAvroSerializer'
        case 'protobuf_schema':
            return 'io.confluent.kafka.serializers.protobuf.KafkaProtobufSerializer'
        default:
            error("❌ Unsupported schema type: ${schemaType}")
    }
}

def getSchemaSubject() {
    if (params.SCHEMA_SUBJECT?.trim()) {
        return params.SCHEMA_SUBJECT.trim()
    } else {
        return "${params.TOPIC_NAME}-value"
    }
}

def createKafkaClientConfig(username, password) {
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
        default:
            securityConfig = """
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="${username}" password="${password}";
"""
            break
    }

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c 'cat > ${env.CLIENT_CONFIG_FILE} << "EOF"
bootstrap.servers=${params.KAFKA_BOOTSTRAP_SERVER}
${securityConfig}
EOF'
    """
}

def validateSchemaRegistry() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                RESPONSE=\$(curl -s -o /dev/null -w "%{http_code}" ${params.SCHEMA_REGISTRY_URL}/subjects 2>/dev/null)
                if [ "\$RESPONSE" = "200" ]; then
                    echo "✅ Schema Registry is accessible at ${params.SCHEMA_REGISTRY_URL}"
                else
                    echo "❌ Schema Registry is not accessible (HTTP \$RESPONSE)"
                    exit 1
                fi
            '
        """
    } catch (Exception e) {
        error("❌ Schema Registry validation failed: ${e.getMessage()}")
    }
}

def prepareSchemaDefinition() {
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            cat > ${env.SCHEMA_FILE} << "SCHEMA_EOF"
${params.SCHEMA_DEFINITION}
SCHEMA_EOF
            echo "✅ Schema definition prepared"
            echo "Schema content preview:"
            head -10 ${env.SCHEMA_FILE}
        '
    """
}

def registerSchema(schemaSubject) {
    try {
        // Prepare the schema payload based on schema type
        def schemaType = params.SCHEMA_TYPE.replace('_SCHEMA', '').toUpperCase()
        def escapedSchema = params.SCHEMA_DEFINITION.replace('"', '\\"').replace('\n', '\\n').replace('\r', '')
        
        def requestBodyMap = [:]
        requestBodyMap.schema = escapedSchema
        
        // Add schemaType field for non-Avro schemas
        if (schemaType != 'AVRO') {
            requestBodyMap.schemaType = schemaType
        }
        
        def requestBody = JsonOutput.toJson(requestBodyMap)
        
        echo "📦 Registering schema for subject: ${schemaSubject}"
        echo "📦 Schema type: ${schemaType}"
        
        def response = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    curl -s -w "\\n%{http_code}" -X POST \\
                    -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
                    --data '${requestBody}' \\
                    ${params.SCHEMA_REGISTRY_URL}/subjects/${schemaSubject}/versions
                '
            """,
            returnStdout: true
        ).trim()

        def lines = response.split('\n')
        def httpCode = lines[-1]
        def responseBody = lines.size() > 1 ? lines[0..-2].join('\n') : ''

        echo "📤 Registration response: ${responseBody}"

        if (httpCode.startsWith('2')) {
            echo "✅ Schema registered successfully (HTTP ${httpCode})"
            if (responseBody.contains('"id"')) {
                def schemaId = responseBody.replaceAll('.*"id":(\\d+).*', '$1')
                echo "✅ Schema ID: ${schemaId}"
            }
        } else {
            error("❌ Schema registration failed (HTTP ${httpCode}): ${responseBody}")
        }
        
    } catch (Exception e) {
        error("❌ Schema registration failed: ${e.getMessage()}")
    }
}

def validateExistingSchema(schemaSubject) {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                echo "Validating existing schema for subject: ${schemaSubject}"
                RESPONSE=\$(curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${schemaSubject}/versions/latest)
                
                if echo "\$RESPONSE" | grep -q "schema"; then
                    SCHEMA_ID=\$(echo "\$RESPONSE" | grep -o \'"id":[0-9]*\' | cut -d: -f2)
                    echo "✅ Schema found with ID: \$SCHEMA_ID"
                else
                    echo "❌ Schema not found for subject: ${schemaSubject}"
                    echo "Response: \$RESPONSE"
                    exit 1
                fi
            '
        """
    } catch (Exception e) {
        error("❌ Schema validation failed: ${e.getMessage()}")
    }
}

def prepareMessageDataFromFile() {
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            if [ -f "${params.INPUT_FILE_PATH}" ]; then
                cp "${params.INPUT_FILE_PATH}" "${env.MESSAGE_DATA_FILE}"
                echo "✅ Message data copied from ${params.INPUT_FILE_PATH}"
            else
                echo "❌ Input file ${params.INPUT_FILE_PATH} not found"
                exit 1
            fi
        '
    """
}

def prepareMessageDataFromParameter() {
    def messageCount = params.MESSAGE_COUNT.toInteger()
    def messageData = params.MESSAGE_DATA.trim()

    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            echo "Preparing ${messageCount} schema-based messages..."
            rm -f "${env.MESSAGE_DATA_FILE}"
            for i in \$(seq 1 ${messageCount}); do
                echo "${messageData}" >> "${env.MESSAGE_DATA_FILE}"
            done
            echo "✅ Message data file prepared with ${messageCount} messages"
            echo "Sample content:"
            head -3 "${env.MESSAGE_DATA_FILE}"
        '
    """
}

def validateMessageAgainstSchema() {
    echo "🔍 Validating message data against schema..."
    // This is a basic validation - in production you might want more sophisticated validation
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T broker bash -c '
            echo "Performing basic JSON format validation..."
            if command -v jq >/dev/null 2>&1; then
                head -1 ${env.MESSAGE_DATA_FILE} | jq . >/dev/null
                if [ \$? -eq 0 ]; then
                    echo "✅ Message data appears to be valid JSON format"
                else
                    echo "❌ Message data is not valid JSON format"
                    exit 1
                fi
            else
                echo "⚠️  jq not available - skipping JSON validation"
            fi
        '
    """
}

def saveProducerResults(result) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Kafka Schema-Based Producer Results
# Generated: ${timestamp}
# Topic: ${params.TOPIC_NAME}
# Schema Type: ${params.SCHEMA_TYPE}
# Schema Subject: ${getSchemaSubject()}
# Message Count: ${params.MESSAGE_COUNT}
# Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security Protocol: ${params.SECURITY_PROTOCOL}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

================================================================================
PRODUCER EXECUTION RESULTS
================================================================================

${result}

================================================================================
SCHEMA CONFIGURATION
================================================================================

Schema Type: ${params.SCHEMA_TYPE}
Schema Subject: ${getSchemaSubject()}
Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}
Auto Register Schema: ${params.AUTO_REGISTER_SCHEMA}
Validate Schema Compatibility: ${params.VALIDATE_SCHEMA_COMPATIBILITY}

Schema Definition:
${params.SCHEMA_DEFINITION}

================================================================================
CONFIGURATION SUMMARY
================================================================================

Topic Name: ${params.TOPIC_NAME}
Message Count: ${params.MESSAGE_COUNT}
Bootstrap Server: ${params.KAFKA_BOOTSTRAP_SERVER}
Security Protocol: ${params.SECURITY_PROTOCOL}
Use File Input: ${params.USE_FILE_INPUT}
Input File Path: ${params.INPUT_FILE_PATH}

================================================================================
"""

    writeFile file: env.PRODUCER_OUTPUT_FILE, text: content
}

def cleanupFiles() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE} ${env.MESSAGE_DATA_FILE} ${env.SCHEMA_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}