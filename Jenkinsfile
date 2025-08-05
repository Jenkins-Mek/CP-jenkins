//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'SUBJECT_NAME', defaultValue: '', description: 'Enter subject name to describe (or leave empty to list all subjects)'),
        booleanParam(name: 'INCLUDE_VERSIONS', defaultValue: true, description: 'Include all versions for each subject')
    ])
])

pipeline {
    agent any

    environment {
        SCHEMA_SUBJECTS_LIST_PATH = '/var/lib/jenkins/workspace/schema-subjects-list.txt'
        SCHEMA_SUBJECTS_LIST_FILE = 'schema-subjects-list.txt'
        SCHEMA_SUBJECT_DESCRIPTION_FILE = 'schema-subject-description.txt'
        SCHEMA_REGISTRY_CONFIG_FILE = '/tmp/schema-registry-client.properties'
    }

    stages {
        stage('Create Schema Registry Configuration') {
            steps {
                script {
                    createSchemaRegistryConfig()
                }
            }
        }

        stage('List Schema Subjects') {
            when {
                expression { !params.SUBJECT_NAME?.trim() }
            }
            steps {
                script {
                    def subjects = listSchemaSubjects()
                    if (!subjects || subjects.isEmpty()) {
                        echo "ℹ️ No schema subjects found in registry."
                        writeFile file: env.SCHEMA_SUBJECTS_LIST_PATH, text: "# No schema subjects found\n# Generated: ${new Date().format('yyyy-MM-dd HH:mm:ss')}\n# Schema Registry: ${params.SCHEMA_REGISTRY_URL}\n\nNo subjects registered in the schema registry."
                        return
                    }

                    echo "📋 Available schema subjects (${subjects.size()}):"
                    subjects.eachWithIndex { subject, i -> echo "  ${i + 1}. ${subject}" }

                    // Get detailed information if requested
                    def subjectDetails = [:]
                    if (params.INCLUDE_VERSIONS) {
                        echo "\n🔍 Fetching version details for each subject..."
                        subjects.each { subject ->
                            subjectDetails[subject] = getSubjectVersions(subject)
                        }
                    }

                    saveSubjectListToFile(subjects, subjectDetails)
                    echo "\n💡 Re-run this job with a SUBJECT_NAME to describe a specific subject."
                }
            }
        }

        stage('Describe Schema Subject') {
            when {
                expression { params.SUBJECT_NAME?.trim() }
            }
            steps {
                script {
                    def inputSubject = params.SUBJECT_NAME.trim()
                    def subjects = listSchemaSubjects()

                    if (!subjects.contains(inputSubject)) {
                        def matches = subjects.findAll { it.toLowerCase().contains(inputSubject.toLowerCase()) }
                        if (matches.size() == 1) {
                            echo "✅ Partial match found: '${matches[0]}'"
                            inputSubject = matches[0]
                        } else if (matches.size() > 1) {
                            echo "❌ Multiple matches found for '${params.SUBJECT_NAME}':"
                            matches.each { echo "  - ${it}" }
                            error("Please provide a more specific subject name.")
                        } else {
                            echo "❌ Subject '${inputSubject}' not found."
                            error("Available subjects:\n" + subjects.take(10).collect { "  - $it" }.join('\n'))
                        }
                    } else {
                        echo "✅ Subject '${inputSubject}' found"
                    }

                    echo "📝 Describing subject: ${inputSubject}"
                    def subjectInfo = describeSchemaSubject(inputSubject)
                    saveSubjectDescriptionToFile(inputSubject, subjectInfo)
                    echo "✅ Subject description saved to ${env.SCHEMA_SUBJECT_DESCRIPTION_FILE}"
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupSchemaRegistryConfig()
            }
        }
        success {
            script {
                // Archive the appropriate file based on which operation was performed
                def fileToArchive = params.SUBJECT_NAME?.trim() ?
                    env.SCHEMA_SUBJECT_DESCRIPTION_FILE :
                    env.SCHEMA_SUBJECTS_LIST_FILE

                archiveArtifacts artifacts: "${fileToArchive}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Artifact '${fileToArchive}' archived successfully."
            }
        }
    }
}

def createSchemaRegistryConfig() {
    sh """
        docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
        exec -T schema-registry bash -c 'cat > ${env.SCHEMA_REGISTRY_CONFIG_FILE} << "EOF"
schema.registry.url=${params.SCHEMA_REGISTRY_URL}
EOF'
    """
}

def listSchemaSubjects() {
    try {
        def subjectsOutput = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    RESPONSE=\$(curl -s ${params.SCHEMA_REGISTRY_URL}/subjects 2>/dev/null)
                    if [ "\$RESPONSE" = "[]" ] || [ -z "\$RESPONSE" ]; then
                        echo ""
                    else
                        echo "\$RESPONSE" | sed "s/\\[//g" | sed "s/\\]//g" | sed "s/\\"//g" | tr "," "\\n" | grep -v "^[[:space:]]*\$"
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        if (!subjectsOutput) {
            return []
        }

        return subjectsOutput.split('\n').findAll { it.trim() != '' }
    } catch (Exception e) {
        echo "⚠️ Warning: Failed to list schema subjects - ${e.getMessage()}"
        return []
    }
}

def getSubjectVersions(subjectName) {
    try {
        def versionsOutput = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${subjectName}/versions 2>/dev/null
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        return versionsOutput
    } catch (Exception e) {
        return "ERROR: Failed to get versions for subject '${subjectName}'"
    }
}



def describeSchemaSubject(subjectName) {
    try {
        // Get latest version and all versions
        def schemaInfoOutput = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    echo "=== Latest Version Info ==="
                    LATEST=\$(curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${subjectName}/versions/latest 2>/dev/null)
                    echo "\$LATEST"
                    echo
                    echo
                    echo "=== All Versions ==="
                    curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${subjectName}/versions 2>/dev/null
                    echo
                    echo
                    echo "=== Subject Compatibility ==="

                    # Check subject-level compatibility first
                    SUBJECT_COMPAT=\$(curl -s -w "%{http_code}" ${params.SCHEMA_REGISTRY_URL}/config/${subjectName} 2>/dev/null)
                    HTTP_CODE=\$(echo "\$SUBJECT_COMPAT" | tail -c 4)

                    if [ "\$HTTP_CODE" = "200" ]; then
                        echo "\$SUBJECT_COMPAT" | head -c -4
                    else
                        echo "Subject-level compatibility: Not configured (using global settings)"
                        echo
                        echo "=== Global Compatibility ==="
                        curl -s ${params.SCHEMA_REGISTRY_URL}/config 2>/dev/null || echo "Unable to retrieve global compatibility settings"
                    fi
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        // Parse and format the schema for better readability
        def formattedOutput = formatSchemaOutput(schemaInfoOutput)
        return formattedOutput

    } catch (Exception e) {
        return "ERROR: Failed to describe subject '${subjectName}' - ${e.getMessage()}"
    }
}

def formatSchemaOutput(rawOutput) {
    try {
        def lines = rawOutput.split('\n')
        def formattedLines = []
        def inLatestVersion = false

        for (line in lines) {
            if (line.contains('=== Latest Version Info ===')) {
                formattedLines.add(line)
                inLatestVersion = true
                continue
            }

            if (line.contains('=== All Versions ===') || line.contains('=== Subject Compatibility ===')) {
                inLatestVersion = false
                formattedLines.add('')
                formattedLines.add(line)
                continue
            }

            // Format the schema JSON in the latest version section
            if (inLatestVersion && line.trim().startsWith('{"subject"')) {
                def schemaInfo = parseSchemaResponse(line)
                formattedLines.addAll(schemaInfo)
            } else {
                formattedLines.add(line)
            }
        }

        return formattedLines.join('\n')

    } catch (Exception e) {
        echo "⚠️ Warning: Failed to format schema output, using raw format - ${e.getMessage()}"
        return rawOutput
    }
}

def parseSchemaResponse(jsonLine) {
    try {
        // Parse the JSON response
        def jsonSlurper = new groovy.json.JsonSlurper()
        def schemaData = jsonSlurper.parseText(jsonLine)

        def formattedLines = []
        formattedLines.add("Subject: ${schemaData.subject}")
        formattedLines.add("Version: ${schemaData.version}")
        formattedLines.add("Schema ID: ${schemaData.id}")

        // Detect schema type
        def schemaType = detectSchemaType(schemaData)
        formattedLines.add("Schema Type: ${schemaType}")
        formattedLines.add("")

        // Format based on schema type
        switch (schemaType.toLowerCase()) {
            case 'avro':
                formattedLines.addAll(formatAvroSchema(schemaData.schema))
                break
            case 'json':
                formattedLines.addAll(formatJsonSchema(schemaData.schema))
                break
            case 'protobuf':
                formattedLines.addAll(formatProtobufSchema(schemaData.schema))
                break
            default:
                formattedLines.addAll(formatGenericSchema(schemaData.schema, schemaType))
                break
        }

        formattedLines.add("")
        formattedLines.add("=" * 80)

        return formattedLines

    } catch (Exception e) {
        echo "Warning: Failed to parse schema response, using original format - ${e.getMessage()}"
        return [jsonLine]
    }
}

def detectSchemaType(schemaData) {
    try {
        def schema = schemaData.schema

        // Check if it has schemaType field (newer Schema Registry versions)
        if (schemaData.schemaType) {
            return schemaData.schemaType
        }

        // Try to detect by schema content
        if (schema.startsWith('syntax = "proto')) {
            return 'PROTOBUF'
        } else if (schema.contains('"$schema"') && schema.contains('json-schema.org')) {
            return 'JSON'
        } else if (schema.startsWith('{') && (schema.contains('"type"') || schema.contains('"fields"'))) {
            return 'AVRO'
        } else {
            return 'UNKNOWN'
        }
    } catch (Exception e) {
        return 'UNKNOWN'
    }
}

def formatAvroSchema(schemaString) {
    try {
        def jsonSlurper = new groovy.json.JsonSlurper()
        def schemaJson = jsonSlurper.parseText(schemaString)

        def lines = []
        lines.add("Avro Schema (Ready to Use):")
        lines.add("=" * 50)

        // Pretty print the JSON schema with proper indentation
        def prettyJson = groovy.json.JsonOutput.prettyPrint(schemaString)
        lines.add(prettyJson)

        lines.add("=" * 50)
        lines.add("")

        // Add quick summary for reference
        lines.add("Quick Summary:")
        lines.add("  Type: ${schemaJson.type}")

        if (schemaJson.name) {
            lines.add("  Name: ${schemaJson.name}")
        }

        if (schemaJson.namespace) {
            lines.add("  Namespace: ${schemaJson.namespace}")
        }

        if (schemaJson.fields) {
            lines.add("  Fields: ${schemaJson.fields.collect { it.name }.join(', ')}")
        } else if (schemaJson.symbols) {
            lines.add("  Enum Values: ${schemaJson.symbols.join(', ')}")
        }

        return lines

    } catch (Exception e) {
        echo "⚠️ Warning: Failed to parse Avro schema - ${e.getMessage()}"
        return formatGenericSchema(schemaString, 'AVRO')
    }
}

def formatJsonSchema(schemaString) {
    try {
        def jsonSlurper = new groovy.json.JsonSlurper()
        def schemaJson = jsonSlurper.parseText(schemaString)

        def lines = []
        lines.add("JSON Schema (Ready to Use):")
        lines.add("=" * 50)

        // Pretty print the JSON schema with proper indentation
        def prettyJson = groovy.json.JsonOutput.prettyPrint(schemaString)
        lines.add(prettyJson)

        lines.add("=" * 50)
        lines.add("")

        // Add quick summary for reference
        lines.add("Quick Summary:")

        if (schemaJson.'$schema') {
            lines.add("  Schema Version: ${schemaJson.'$schema'}")
        }

        if (schemaJson.title) {
            lines.add("  Title: ${schemaJson.title}")
        }

        if (schemaJson.type) {
            lines.add("  Type: ${schemaJson.type}")
        }

        if (schemaJson.properties) {
            def propertyNames = schemaJson.properties.keySet().take(10) // Show first 10 properties
            lines.add("  Properties: ${propertyNames.join(', ')}${schemaJson.properties.size() > 10 ? '...' : ''}")
        }

        if (schemaJson.required) {
            lines.add("  Required Fields: ${schemaJson.required.join(', ')}")
        }

        return lines

    } catch (Exception e) {
        echo "Warning: Failed to parse JSON schema - ${e.getMessage()}"
        return formatGenericSchema(schemaString, 'JSON')
    }
}

def formatProtobufSchema(schemaString) {
    def lines = []
    lines.add("Protocol Buffer Schema:")
    lines.add("")

    // Split into lines and format
    def schemaLines = schemaString.split('\n')
    def inMessage = false
    def indent = ""

    for (line in schemaLines) {
        def trimmedLine = line.trim()

        if (trimmedLine.startsWith('syntax')) {
            lines.add("${trimmedLine}")
        } else if (trimmedLine.startsWith('package')) {
            lines.add("${trimmedLine}")
        } else if (trimmedLine.startsWith('import')) {
            lines.add("${trimmedLine}")
        } else if (trimmedLine.startsWith('message')) {
            lines.add("")
            lines.add("${trimmedLine}")
            inMessage = true
            indent = "  "
        } else if (trimmedLine.startsWith('enum')) {
            lines.add("")
            lines.add("${trimmedLine}")
            inMessage = true
            indent = "  "
        } else if (trimmedLine == '}') {
            lines.add("${indent}}")
            inMessage = false
            indent = ""
        } else if (trimmedLine && inMessage) {
            lines.add("${indent}${trimmedLine}")
        } else if (trimmedLine) {
            lines.add(trimmedLine)
        }
    }

    return lines
}

def formatGenericSchema(schemaString, schemaType) {
    def lines = []
    lines.add("${schemaType} Schema:")
    lines.add("")
    lines.add("Raw Schema Content:")
    lines.add("┌" + "─" * 78 + "┐")

    // Split long lines and add proper formatting
    def schemaLines = schemaString.split('\n')
    for (line in schemaLines) {
        if (line.length() > 76) {
            // Break long lines
            def words = line.split(' ')
            def currentLine = ""
            for (word in words) {
                if ((currentLine + word).length() > 76) {
                    lines.add("│ ${currentLine.padRight(76)} │")
                    currentLine = word + " "
                } else {
                    currentLine += word + " "
                }
            }
            if (currentLine.trim()) {
                lines.add("│ ${currentLine.trim().padRight(76)} │")
            }
        } else {
            lines.add("│ ${line.padRight(76)} │")
        }
    }
    
    lines.add("└" + "─" * 78 + "┘")
    
    return lines
}

def formatFieldType(fieldType) {
    if (fieldType instanceof String) {
        return fieldType
    } else if (fieldType instanceof List) {
        // Handle union types
        def types = fieldType.collect { 
            it instanceof String ? it : (it.type ?: it.toString())
        }
        return "Union[${types.join(', ')}]"
    } else if (fieldType instanceof Map) {
        // Handle complex types
        if (fieldType.type == 'array') {
            return "Array[${formatFieldType(fieldType.items)}]"
        } else if (fieldType.type == 'map') {
            return "Map[${formatFieldType(fieldType.values)}]"
        } else if (fieldType.type == 'record') {
            return "Record[${fieldType.name}]"
        } else if (fieldType.type == 'enum') {
            return "Enum[${fieldType.symbols?.join(', ')}]"
        } else {
            return fieldType.type ?: fieldType.toString()
        }
    } else {
        return fieldType.toString()
    }
}

def saveSubjectDescriptionToFile(subjectName, subjectInfo) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def textContent = """# Schema Subject Description
# Generated: ${timestamp}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}
# Subject: ${subjectName}

================================================================================
Subject: ${subjectName}
================================================================================
${subjectInfo}

"""

    writeFile file: env.SCHEMA_SUBJECT_DESCRIPTION_FILE, text: textContent
}

def saveSubjectListToFile(subjects, subjectDetails = [:]) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def textContent = """# Schema Registry Subjects List
# Generated: ${timestamp}
# Total subjects: ${subjects.size()}
# Schema Registry URL: ${params.SCHEMA_REGISTRY_URL}
# Include versions: ${params.INCLUDE_VERSIONS}

"""

    if (subjects.isEmpty()) {
        textContent += "No schema subjects found in the registry.\n"
    } else {
        textContent += "#Available Schema Subjects:\n"
        textContent += "#"+"=" * 50 + "\n\n"

        subjects.eachWithIndex { subject, index ->
            textContent += "${subject}"

            if (params.INCLUDE_VERSIONS && subjectDetails.containsKey(subject)) {
                textContent += "${subjectDetails[subject]}\n"
            } else textContent += "\n"
        }

        textContent += "\n" +"#"+"=" * 50 + "\n"
        textContent += "\n#To get detailed information about a specific subject, re-run with SUBJECT_NAME parameter.\n"
    }

    writeFile file: env.SCHEMA_SUBJECTS_LIST_PATH, text: textContent
}


def cleanupSchemaRegistryConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c 'rm -f ${env.SCHEMA_REGISTRY_CONFIG_FILE}' 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}