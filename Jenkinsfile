//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'SUBJECT_NAME', defaultValue: '', description: 'Subject name for the schema (required)'),
        choice(name: 'SCHEMA_TYPE', choices: ['AVRO', 'JSON', 'PROTOBUF'], description: 'Schema type'),
        text(name: 'SCHEMA_CONTENT', defaultValue: '', description: 'Schema content (JSON for Avro/JSON Schema, .proto for Protobuf)'),
        choice(name: 'COMPATIBILITY', choices: ['', 'BACKWARD', 'BACKWARD_TRANSITIVE', 'FORWARD', 'FORWARD_TRANSITIVE', 'FULL', 'FULL_TRANSITIVE', 'NONE'], description: 'Compatibility level (optional - leave empty to use global/existing)'),
        booleanParam(name: 'DRY_RUN', defaultValue: false, description: 'Validate schema without registering'),
        booleanParam(name: 'FORCE_REGISTER', defaultValue: false, description: 'Force registration even if schema already exists')
    ])
])

pipeline {
    agent any

    environment {
        SCHEMA_REGISTRY_CONFIG_FILE = '/tmp/schema-registry-client.properties'
        TEMP_SCHEMA_FILE = '/tmp/temp-schema.json'
        REGISTRATION_RESULT_FILE = 'schema-registration-result.txt'
    }

    stages {
        stage('Validate Input Parameters') {
            steps {
                script {
                    validateInputParameters()
                }
            }
        }

        stage('Create Schema Registry Configuration') {
            steps {
                script {
                    createSchemaRegistryConfig()
                }
            }
        }

        stage('Validate Schema Content') {
            steps {
                script {
                    validateSchemaContent()
                }
            }
        }

        stage('Check Existing Schema') {
            when {
                expression { !params.FORCE_REGISTER }
            }
            steps {
                script {
                    checkExistingSchema()
                }
            }
        }

        stage('Set Subject Compatibility') {
            when {
                expression { params.COMPATIBILITY?.trim() }
            }
            steps {
                script {
                    setSubjectCompatibility()
                }
            }
        }

        stage('Dry Run - Validate Schema') {
            when {
                expression { params.DRY_RUN }
            }
            steps {
                script {
                    performDryRun()
                }
            }
        }

        stage('Register Schema') {
            when {
                expression { !params.DRY_RUN }
            }
            steps {
                script {
                    registerSchema()
                }
            }
        }

        stage('Verify Registration') {
            when {
                expression { !params.DRY_RUN }
            }
            steps {
                script {
                    verifyRegistration()
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupTempFiles()
            }
        }
        success {
            script {
                archiveArtifacts artifacts: "${env.REGISTRATION_RESULT_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Registration result archived successfully."
            }
        }
        failure {
            script {
                echo "❌ Schema registration failed. Check the logs for details."
            }
        }
    }
}

def validateInputParameters() {
    echo "🔍 Validating input parameters..."
    
    if (!params.SUBJECT_NAME?.trim()) {
        error("❌ SUBJECT_NAME is required")
    }
    
    if (!params.SCHEMA_CONTENT?.trim()) {
        error("❌ SCHEMA_CONTENT is required")
    }
    
    if (!params.SCHEMA_TYPE?.trim()) {
        error("❌ SCHEMA_TYPE is required")
    }
    
    // Validate schema type
    def validTypes = ['AVRO', 'JSON', 'PROTOBUF']
    if (!validTypes.contains(params.SCHEMA_TYPE)) {
        error("❌ Invalid SCHEMA_TYPE. Must be one of: ${validTypes.join(', ')}")
    }
    
    echo "✅ Input parameters validated successfully"
    echo "   Subject: ${params.SUBJECT_NAME}"
    echo "   Schema Type: ${params.SCHEMA_TYPE}"
    echo "   Compatibility: ${params.COMPATIBILITY ?: 'Not specified (using default)'}"
    echo "   Dry Run: ${params.DRY_RUN}"
    echo "   Force Register: ${params.FORCE_REGISTER}"
}

def createSchemaRegistryConfig() {
    echo "⚙️ Creating Schema Registry configuration..."
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T schema-registry bash -c 'cat > ${env.SCHEMA_REGISTRY_CONFIG_FILE} << "EOF"
schema.registry.url=${params.SCHEMA_REGISTRY_URL}
EOF'
    """
    echo "✅ Schema Registry configuration created"
}

def validateSchemaContent() {
    echo "🔍 Validating schema content..."
    
    try {
        // Create temporary schema file based on type
        def schemaContent = prepareSchemaContent()
        
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c 'cat > ${env.TEMP_SCHEMA_FILE} << "EOF"
${schemaContent}
EOF'
        """
        
        // Perform basic validation based on schema type
        switch (params.SCHEMA_TYPE) {
            case 'AVRO':
                validateAvroSchema()
                break
            case 'JSON':
                validateJsonSchema()
                break
            case 'PROTOBUF':
                validateProtobufSchema()
                break
        }
        
        echo "✅ Schema content validation passed"
    } catch (Exception e) {
        error("❌ Schema validation failed: ${e.getMessage()}")
    }
}

def prepareSchemaContent() {
    def content = params.SCHEMA_CONTENT.trim()
    
    if (params.SCHEMA_TYPE == 'AVRO' || params.SCHEMA_TYPE == 'JSON') {
        // For AVRO and JSON schemas, validate JSON format
        try {
            def jsonSlurper = new groovy.json.JsonSlurper()
            def parsed = jsonSlurper.parseText(content)
            // Return the original content since JsonBuilder is not allowed
            return content
        } catch (Exception e) {
            error("❌ Invalid JSON format for ${params.SCHEMA_TYPE} schema: ${e.getMessage()}")
        }
    } else if (params.SCHEMA_TYPE == 'PROTOBUF') {
        // For Protobuf, return as-is
        return content
    }
    
    return content
}

def validateAvroSchema() {
    echo "🔍 Validating Avro schema format..."
    
    def result = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                # Basic Avro schema validation
                SCHEMA_CONTENT=\$(cat ${env.TEMP_SCHEMA_FILE})
                
                # Check if it has required Avro fields
                if echo "\$SCHEMA_CONTENT" | jq -e ".type" > /dev/null 2>&1; then
                    echo "✅ Avro schema has required type field"
                    
                    # Check for record type specific fields
                    SCHEMA_TYPE=\$(echo "\$SCHEMA_CONTENT" | jq -r ".type")
                    if [ "\$SCHEMA_TYPE" = "record" ]; then
                        if echo "\$SCHEMA_CONTENT" | jq -e ".name" > /dev/null 2>&1 && echo "\$SCHEMA_CONTENT" | jq -e ".fields" > /dev/null 2>&1; then
                            echo "✅ Avro record schema has required name and fields"
                        else
                            echo "❌ Avro record schema missing name or fields"
                            exit 1
                        fi
                    fi
                else
                    echo "❌ Invalid Avro schema: missing type field"
                    exit 1
                fi
            '
        """,
        returnStatus: true
    )
    
    if (result != 0) {
        error("❌ Avro schema validation failed")
    }
}

def validateJsonSchema() {
    echo "🔍 Validating JSON schema format..."
    
    def result = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                # Basic JSON schema validation
                SCHEMA_CONTENT=\$(cat ${env.TEMP_SCHEMA_FILE})
                
                # Check if it has JSON schema indicators
                if echo "\$SCHEMA_CONTENT" | jq -e "." > /dev/null 2>&1; then
                    echo "✅ Valid JSON format"
                    
                    # Check for JSON Schema specific fields (optional but recommended)
                    if echo "\$SCHEMA_CONTENT" | jq -e ".type" > /dev/null 2>&1 || echo "\$SCHEMA_CONTENT" | jq -e ".\\\$schema" > /dev/null 2>&1; then
                        echo "✅ JSON Schema appears to have valid structure"
                    else
                        echo "⚠️ Warning: JSON Schema might be missing type or \\\$schema field"
                    fi
                else
                    echo "❌ Invalid JSON format"
                    exit 1
                fi
            '
        """,
        returnStatus: true
    )
    
    if (result != 0) {
        error("❌ JSON schema validation failed")
    }
}

def validateProtobufSchema() {
    echo "🔍 Validating Protobuf schema format..."
    
    def result = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                # Basic Protobuf schema validation
                SCHEMA_CONTENT=\$(cat ${env.TEMP_SCHEMA_FILE})
                
                # Check for basic Protobuf syntax
                if echo "\$SCHEMA_CONTENT" | grep -q "syntax.*=.*\"proto"; then
                    echo "✅ Protobuf schema has syntax declaration"
                else
                    echo "⚠️ Warning: Protobuf schema might be missing syntax declaration"
                fi
                
                # Check for message definition
                if echo "\$SCHEMA_CONTENT" | grep -q "message.*{"; then
                    echo "✅ Protobuf schema has message definition"
                else
                    echo "❌ Protobuf schema missing message definition"
                    exit 1
                fi
            '
        """,
        returnStatus: true
    )
    
    if (result != 0) {
        error("❌ Protobuf schema validation failed")
    }
}

def checkExistingSchema() {
    echo "🔍 Checking if subject already exists..."
    
    try {
        def existingSchema = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s -w "%{http_code}" ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}/versions/latest 2>/dev/null)
                    HTTP_CODE=\$(echo "\$RESPONSE" | tail -c 4)
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        echo "\$RESPONSE" | head -c -4
                        exit 0
                    elif [ "\$HTTP_CODE" = "404" ]; then
                        echo "SUBJECT_NOT_FOUND"
                        exit 0
                    else
                        echo "ERROR: HTTP \$HTTP_CODE"
                        exit 1
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        if (existingSchema == "SUBJECT_NOT_FOUND") {
            echo "✅ Subject '${params.SUBJECT_NAME}' does not exist - ready for registration"
        } else if (existingSchema.startsWith("ERROR:")) {
            error("❌ Failed to check existing schema: ${existingSchema}")
        } else {
            // Parse existing schema info
            def jsonSlurper = new groovy.json.JsonSlurper()
            def schemaInfo = jsonSlurper.parseText(existingSchema)
            
            echo "⚠️ Subject '${params.SUBJECT_NAME}' already exists:"
            echo "   Current Version: ${schemaInfo.version}"
            echo "   Schema ID: ${schemaInfo.id}"
            
            if (!params.FORCE_REGISTER) {
                error("❌ Subject already exists. Use FORCE_REGISTER=true to register a new version or choose a different subject name.")
            } else {
                echo "🔸 FORCE_REGISTER enabled - will attempt to register new version"
            }
        }
    } catch (Exception e) {
        if (e.getMessage().contains("Subject already exists")) {
            throw e
        } else {
            echo "⚠️ Warning: Could not check existing schema - ${e.getMessage()}"
            echo "🔸 Proceeding with registration attempt..."
        }
    }
}

def setSubjectCompatibility() {
    echo "⚙️ Setting subject compatibility to ${params.COMPATIBILITY}..."
    
    try {
        def result = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s -w "%{http_code}" -X PUT \\
                        -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
                        -d "{\"compatibility\":\"${params.COMPATIBILITY}\"}" \\
                        ${params.SCHEMA_REGISTRY_URL}/config/${params.SUBJECT_NAME} 2>/dev/null)
                    
                    HTTP_CODE=\$(echo "\$RESPONSE" | tail -c 4)
                    RESPONSE_BODY=\$(echo "\$RESPONSE" | head -c -4)
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        echo "✅ Compatibility set to ${params.COMPATIBILITY}"
                        echo "\$RESPONSE_BODY"
                    else
                        echo "❌ Failed to set compatibility: HTTP \$HTTP_CODE"
                        echo "\$RESPONSE_BODY"
                        exit 1
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        
        echo result
    } catch (Exception e) {
        error("❌ Failed to set subject compatibility: ${e.getMessage()}")
    }
}

def performDryRun() {
    echo "🧪 Performing dry run - validating schema compatibility..."
    
    def schemaPayload = buildSchemaPayload()
    
    try {
        def result = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s -w "%{http_code}" -X POST \\
                        -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
                        -d "${schemaPayload}" \\
                        ${params.SCHEMA_REGISTRY_URL}/compatibility/subjects/${params.SUBJECT_NAME}/versions/latest 2>/dev/null)
                    
                    HTTP_CODE=\$(echo "\$RESPONSE" | tail -c 4)
                    RESPONSE_BODY=\$(echo "\$RESPONSE" | head -c -4)
                    
                    echo "HTTP Code: \$HTTP_CODE"
                    echo "Response: \$RESPONSE_BODY"
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        IS_COMPATIBLE=\$(echo "\$RESPONSE_BODY" | jq -r ".is_compatible" 2>/dev/null || echo "unknown")
                        if [ "\$IS_COMPATIBLE" = "true" ]; then
                            echo "✅ Schema is compatible"
                            exit 0
                        else
                            echo "❌ Schema is not compatible"
                            exit 1
                        fi
                    elif [ "\$HTTP_CODE" = "404" ]; then
                        echo "✅ New subject - no compatibility issues"
                        exit 0
                    else
                        echo "❌ Compatibility check failed"
                        exit 1
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        
        echo result
        
        // Save dry run results
        saveDryRunResults(result)
        
    } catch (Exception e) {
        error("❌ Dry run failed: ${e.getMessage()}")
    }
}

def registerSchema() {
    echo "📝 Registering schema for subject '${params.SUBJECT_NAME}'..."
    
    def schemaPayload = buildSchemaPayload()
    
    try {
        def result = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s -w "%{http_code}" -X POST \\
                        -H "Content-Type: application/vnd.schemaregistry.v1+json" \\
                        -d "${schemaPayload}" \\
                        ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}/versions 2>/dev/null)
                    
                    HTTP_CODE=\$(echo "\$RESPONSE" | tail -c 4)
                    RESPONSE_BODY=\$(echo "\$RESPONSE" | head -c -4)
                    
                    echo "HTTP Code: \$HTTP_CODE"
                    echo "Response: \$RESPONSE_BODY"
                    
                    if [ "\$HTTP_CODE" = "200" ]; then
                        SCHEMA_ID=\$(echo "\$RESPONSE_BODY" | jq -r ".id" 2>/dev/null)
                        echo "✅ Schema registered successfully with ID: \$SCHEMA_ID"
                        exit 0
                    else
                        echo "❌ Schema registration failed"
                        exit 1
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        
        echo result
        
        // Extract schema ID for verification
        def lines = result.split('\n')
        def schemaId = null
        for (line in lines) {
            if (line.contains('Schema registered successfully with ID:')) {
                schemaId = line.split('ID: ')[1]
                break
            }
        }
        
        if (schemaId) {
            env.REGISTERED_SCHEMA_ID = schemaId
            echo "🆔 Registered Schema ID: ${schemaId}"
        }
        
    } catch (Exception e) {
        error("❌ Schema registration failed: ${e.getMessage()}")
    }
}

def verifyRegistration() {
    echo "🔍 Verifying schema registration..."
    
    try {
        def result = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}/versions/latest 2>/dev/null)
                    echo "\$RESPONSE"
                    
                    # Verify schema ID matches if we have it
                    if [ -n "${env.REGISTERED_SCHEMA_ID ?: ''}" ]; then
                        RETRIEVED_ID=\$(echo "\$RESPONSE" | jq -r ".id" 2>/dev/null)
                        if [ "\$RETRIEVED_ID" = "${env.REGISTERED_SCHEMA_ID ?: ''}" ]; then
                            echo "✅ Schema ID verification successful"
                        else
                            echo "⚠️ Schema ID mismatch: expected ${env.REGISTERED_SCHEMA_ID ?: ''}, got \$RETRIEVED_ID"
                        fi
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()
        
        // Parse and display verification results
        def lines = result.split('\n')
        def jsonLine = lines.find { it.trim().startsWith('{') }
        
        if (jsonLine) {
            def jsonSlurper = new groovy.json.JsonSlurper()
            def schemaInfo = jsonSlurper.parseText(jsonLine)
            
            echo "✅ Registration verification successful:"
            echo "   Subject: ${schemaInfo.subject}"
            echo "   Version: ${schemaInfo.version}"
            echo "   Schema ID: ${schemaInfo.id}"
            echo "   Schema Type: ${params.SCHEMA_TYPE}"
            
            // Save verification results
            saveRegistrationResults(schemaInfo)
        }
        
    } catch (Exception e) {
        echo "⚠️ Warning: Could not verify registration - ${e.getMessage()}"
    }
}

def buildSchemaPayload() {
    def schemaContent = params.SCHEMA_CONTENT.trim()
    
    // Escape quotes and newlines for JSON payload
    def escapedSchema = schemaContent
        .replace('\\', '\\\\')
        .replace('"', '\\"')
        .replace('\n', '\\n')
        .replace('\r', '\\r')
        .replace('\t', '\\t')
    
    def payload = """{"schemaType":"${params.SCHEMA_TYPE}","schema":"${escapedSchema}"}"""
    
    return payload
}

def saveDryRunResults(results) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Schema Registration Dry Run Results
# Generated: ${timestamp}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}
# Subject: ${params.SUBJECT_NAME}
# Schema Type: ${params.SCHEMA_TYPE}

================================================================================
DRY RUN RESULTS
================================================================================

${results}

Schema Content:
================================================================================
${params.SCHEMA_CONTENT}

"""

    writeFile file: env.REGISTRATION_RESULT_FILE, text: content
    echo "📄 Dry run results saved to ${env.REGISTRATION_RESULT_FILE}"
}

def saveRegistrationResults(schemaInfo) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def content = """# Schema Registration Results
# Generated: ${timestamp}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

================================================================================
REGISTRATION SUCCESSFUL
================================================================================

Subject: ${schemaInfo.subject}
Version: ${schemaInfo.version}
Schema ID: ${schemaInfo.id}
Schema Type: ${params.SCHEMA_TYPE}
Compatibility: ${params.COMPATIBILITY ?: 'Default'}

Schema Content:
================================================================================
${params.SCHEMA_CONTENT}

Registration Details:
================================================================================
- Registration completed at: ${timestamp}
- Force Register: ${params.FORCE_REGISTER}
- Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}

"""

    writeFile file: env.REGISTRATION_RESULT_FILE, text: content
    echo "📄 Registration results saved to ${env.REGISTRATION_RESULT_FILE}"
}

def cleanupTempFiles() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c 'rm -f ${env.SCHEMA_REGISTRY_CONFIG_FILE} ${env.TEMP_SCHEMA_FILE}' 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
        echo "⚠️ Warning: Cleanup failed - ${e.getMessage()}"
    }
}