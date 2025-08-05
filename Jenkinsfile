properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Topic name (e.g., test-topic, user-topic)'),
        choice(name: 'SCHEMA_FOR', choices: ['value', 'key'], description: 'Schema for key or value'),
        string(name: 'CUSTOM_SUBJECT_NAME', defaultValue: '', description: 'Custom subject name (optional - if empty, will use {topic-name}-{key|value})'),
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
                    if (!params.TOPIC_NAME?.trim()) {
                        error("TOPIC_NAME is required")
                    }
                    if (!params.SCHEMA_CONTENT?.trim()) {
                        error("SCHEMA_CONTENT is required")
                    }

                    // Generate subject name - use custom if provided, otherwise use standard naming
                    if (params.CUSTOM_SUBJECT_NAME?.trim()) {
                        env.SUBJECT_NAME = params.CUSTOM_SUBJECT_NAME.trim()
                        echo "📋 Using custom subject name: ${env.SUBJECT_NAME}"
                    } else {
                        env.SUBJECT_NAME = "${params.TOPIC_NAME}-${params.SCHEMA_FOR}"
                        echo "📋 Using standard subject name: ${env.SUBJECT_NAME}"
                    }

                    echo "✅ Input validation passed"
                    echo "📋 Topic: ${params.TOPIC_NAME}"
                    echo "📋 Subject: ${env.SUBJECT_NAME}"
                    echo "📝 Schema Type: ${params.SCHEMA_TYPE}"
                    echo "🔑 Schema For: ${params.SCHEMA_FOR}"
                }
            }
        }

        stage('Check Existing Schema') {
            steps {
                script {
                    // Check if schema already exists
                    def checkResponse = sh(
                        script: """
                            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                            exec -T schema-registry bash -c '
                                curl -s -w "\\n%{http_code}" \\
                                ${params.SCHEMA_REGISTRY_URL}/subjects/${env.SUBJECT_NAME}/versions/latest
                            '
                        """,
                        returnStdout: true
                    ).trim()

                    def checkLines = checkResponse.split('\n')
                    def checkHttpCode = checkLines[-1]
                    def checkResponseBody = checkLines.size() > 1 ? checkLines[0..-2].join('\n') : ''

                    if (checkHttpCode == '200') {
                        echo "ℹ️ Schema already exists for subject: ${env.SUBJECT_NAME}"
                        echo "📋 Existing schema: ${checkResponseBody}"
                        env.SCHEMA_EXISTS = 'true'
                    } else if (checkHttpCode == '404') {
                        echo "📝 No existing schema found for subject: ${env.SUBJECT_NAME}"
                        env.SCHEMA_EXISTS = 'false'
                    } else {
                        echo "⚠️ Unable to check existing schema (HTTP ${checkHttpCode}): ${checkResponseBody}"
                        env.SCHEMA_EXISTS = 'unknown'
                    }
                }
            }
        }

        stage('Register Schema') {
            steps {
                script {
                    echo "${params.SCHEMA_CONTENT}"
                    def escapedSchema = params.SCHEMA_CONTENT.replaceAll(/(?<!\\)"/, '\\"').replace('\n', '\\n').replace('\r', '')

                    def requestBody
                    if (params.SCHEMA_TYPE == 'AVRO') {
                        requestBody = """{"schema": "${escapedSchema}"}"""
                    } else {
                        requestBody = """{"schemaType": "${params.SCHEMA_TYPE}", "schema": "${escapedSchema}"}"""
                    }
                    echo "${escapedSchema}"
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
                        if (env.SCHEMA_EXISTS == 'true') {
                            echo "✅ Schema was already registered - returned existing version (HTTP ${httpCode})"
                        } else {
                            echo "✅ New schema registered successfully (HTTP ${httpCode})"
                        }
                    } else {
                        // Common error cases
                        if (httpCode == '409') {
                            echo "⚠️ Schema conflict (HTTP 409) - schema may be incompatible with existing versions"
                        } else if (httpCode == '422') {
                            echo "⚠️ Schema validation failed (HTTP 422) - invalid schema format"
                        }
                        error("❌ Schema registration failed (HTTP ${httpCode}): ${responseBody}")
                    }
                }
            }
        }

        stage('Verify Registration') {
            steps {
                script {
                    // Inline the verifySchemaRegistration function
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