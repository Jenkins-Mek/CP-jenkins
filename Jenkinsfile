properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'SUBJECT_NAME', defaultValue: '', description: 'Subject name (e.g., user-topic-value)'),
        choice(name: 'SCHEMA_TYPE', choices: ['AVRO', 'JSON', 'PROTOBUF'], description: 'Schema type'),
        text(name: 'SCHEMA_CONTENT', defaultValue: '', description: 'Schema content (JSON for Avro/JSON, proto definition for Protobuf)')
    ])
])

pipeline {
    agent any

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.SUBJECT_NAME?.trim()) {
                        error("SUBJECT_NAME is required")
                    }
                    if (!params.SCHEMA_CONTENT?.trim()) {
                        error("SCHEMA_CONTENT is required")
                    }
                    echo "✅ Input validation passed"
                    echo "📋 Subject: ${params.SUBJECT_NAME}"
                    echo "📝 Schema Type: ${params.SCHEMA_TYPE}"
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
    }

    post {
        success {
            echo "✅ Schema registered successfully for subject: ${params.SUBJECT_NAME}"
        }
        failure {
            echo "❌ Schema registration failed for subject: ${params.SUBJECT_NAME}"
        }
    }
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
                ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}/versions
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
                curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}/versions/latest
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