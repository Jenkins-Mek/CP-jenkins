properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Topic name (e.g., test-topic, user-topic)'),
        choice(name: 'SCHEMA_FOR', choices: ['value', 'key'], description: 'Schema for key or value'),
        choice(name: 'SCHEMA_TYPE', choices: ['AVRO', 'JSON', 'PROTOBUF'], description: 'Schema type'),
        text(name: 'SCHEMA_CONTENT', defaultValue: '', description: 'Schema content (JSON for Avro/JSON, proto definition for Protobuf)')
    ])
])

pipeline {
    agent any

    stages {
        stage('List Available Topics') {
            steps {
                script {
                    listTopics()
                }
            }
        }

        stage('Validate Input') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim()) {
                        error("TOPIC_NAME is required")
                    }
                    if (!params.SCHEMA_CONTENT?.trim()) {
                        error("SCHEMA_CONTENT is required")
                    }
                    
                    // Generate subject name from topic and schema type
                    env.SUBJECT_NAME = "${params.TOPIC_NAME}-${params.SCHEMA_FOR}"
                    
                    echo "✅ Input validation passed"
                    echo "📋 Topic: ${params.TOPIC_NAME}"
                    echo "📋 Subject: ${env.SUBJECT_NAME}"
                    echo "📝 Schema Type: ${params.SCHEMA_TYPE}"
                    echo "🔑 Schema For: ${params.SCHEMA_FOR}"
                }
            }
        }

        stage('Check Topic Exists') {
            steps {
                script {
                    checkTopicExists()
                }
            }
        }

        stage('Register Schema') {
            steps {
                script {
                    registerSchema()
                }
            }
        }

        stage('Verify Registration') {
            steps {
                script {
                    verifySchemaRegistration()
                }
            }
        }

        stage('Show Consumer Commands') {
            steps {
                script {
                    showConsumerCommands()
                }
            }
        }
    }

    post {
        success {
            echo "✅ Schema registered successfully for subject: ${env.SUBJECT_NAME}"
        }
        failure {
            echo "❌ Schema registration failed for subject: ${env.SUBJECT_NAME}"
        }
    }
}

def listTopics() {
    echo "📋 Listing available topics..."
    def topics = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c '
                kafka-topics --list --bootstrap-server localhost:9092
            '
        """,
        returnStdout: true
    ).trim()
    
    echo "Available topics:"
    topics.split('\n').each { topic ->
        echo "  - ${topic}"
    }
}

def checkTopicExists() {
    echo "🔍 Checking if topic '${params.TOPIC_NAME}' exists..."
    def topics = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c '
                kafka-topics --list --bootstrap-server localhost:9092
            '
        """,
        returnStdout: true
    ).trim()
    
    if (!topics.split('\n').contains(params.TOPIC_NAME)) {
        echo "⚠️ Topic '${params.TOPIC_NAME}' does not exist. Creating topic..."
        createTopic()
    } else {
        echo "✅ Topic '${params.TOPIC_NAME}' exists"
    }
}

def createTopic() {
    def response = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c '
                kafka-topics --create --topic ${params.TOPIC_NAME} --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
            '
        """,
        returnStdout: true
    ).trim()
    
    echo "📝 Topic creation response: ${response}"
    echo "✅ Topic '${params.TOPIC_NAME}' created successfully"
}

def registerSchema() {
    def escapedSchema = params.SCHEMA_CONTENT.replace('"', '\\"').replace('\n', '\\n').replace('\r', '')
    
    def requestBody
    if (params.SCHEMA_TYPE == 'AVRO') {
        requestBody = """{"schema": "${escapedSchema}"}"""
    } else {
        requestBody = """{"schemaType": "${params.SCHEMA_TYPE}", "schema": "${escapedSchema}"}"""
    }

    def response = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                curl -s -w "\\n%{http_code}" -X POST \\
                -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
                --data '"'"'${requestBody}'"'"' \\
                ${params.SCHEMA_REGISTRY_URL}/subjects/${env.SUBJECT_NAME}/versions
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
    } else {
        error("❌ Schema registration failed (HTTP ${httpCode}): ${responseBody}")
    }
}

def verifySchemaRegistration() {
    def response = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${env.SUBJECT_NAME}/versions/latest
            '
        """,
        returnStdout: true
    ).trim()

    echo "🔍 Latest schema version: ${response}"
    
    if (response.contains('"subject"') && response.contains('"version"')) {
        def jsonSlurper = new groovy.json.JsonSlurper()
        try {
            def schemaInfo = jsonSlurper.parseText(response)
            echo "✅ Verification successful:"
            echo "   Subject: ${schemaInfo.subject}"
            echo "   Version: ${schemaInfo.version}"
            echo "   Schema ID: ${schemaInfo.id}"
        } catch (Exception e) {
            echo "⚠️ Schema registered but verification parsing failed: ${e.getMessage()}"
        }
    } else {
        error("❌ Schema verification failed: ${response}")
    }
}

def showConsumerCommands() {
    echo "\n🚀 Consumer Commands for your registered schema:"
    echo "=" * 60
    
    if (params.SCHEMA_TYPE == 'AVRO') {
        echo "📝 Avro Consumer Command:"
        echo """
docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
    exec -T broker bash -c "
        kafka-avro-console-consumer --bootstrap-server localhost:9092 \\
        --topic ${params.TOPIC_NAME} --from-beginning \\
        --property schema.registry.url=${params.SCHEMA_REGISTRY_URL}
    "
        """
    } else if (params.SCHEMA_TYPE == 'JSON') {
        echo "📝 JSON Schema Consumer Command:"
        echo """
docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
    exec -T broker bash -c "
        kafka-json-schema-console-consumer --bootstrap-server localhost:9092 \\
        --topic ${params.TOPIC_NAME} --from-beginning \\
        --property schema.registry.url=${params.SCHEMA_REGISTRY_URL}
    "
        """
    } else if (params.SCHEMA_TYPE == 'PROTOBUF') {
        echo "📝 Protobuf Consumer Command:"
        echo """
docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
    exec -T broker bash -c "
        kafka-protobuf-console-consumer --bootstrap-server localhost:9092 \\
        --topic ${params.TOPIC_NAME} --from-beginning \\
        --property schema.registry.url=${params.SCHEMA_REGISTRY_URL}
    "
        """
    }
    
    echo "\n📝 Regular Console Consumer (without schema validation):"
    echo """
docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
    exec -T broker bash -c "
        kafka-console-consumer --bootstrap-server localhost:9092 \\
        --topic ${params.TOPIC_NAME} --from-beginning
    "
    """
    
    echo "=" * 60
}