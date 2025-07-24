//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'SUBJECT_NAME', defaultValue: '', description: 'Enter subject name to describe (or leave empty to list all subjects)'),
        booleanParam(name: 'INCLUDE_VERSIONS', defaultValue: false, description: 'Include all versions for each subject')
    ])
])

pipeline {
    agent any

    environment {
        SCHEMA_SUBJECTS_FILE = 'schema-subjects-list.txt'
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
                        writeFile file: env.SCHEMA_SUBJECTS_FILE, text: "# No schema subjects found\n# Generated: ${new Date().format('yyyy-MM-dd HH:mm:ss')}\n# Schema Registry: ${params.SCHEMA_REGISTRY_URL}\n\nNo subjects registered in the schema registry."
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
                    echo "✅ Subject description saved to ${env.SCHEMA_SUBJECTS_FILE}"
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
                archiveArtifacts artifacts: "${env.SCHEMA_SUBJECTS_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Artifact '${env.SCHEMA_SUBJECTS_FILE}' archived successfully."
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
        // Get latest version
        def latestVersionOutput = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T schema-registry bash -c '
                    echo "=== Latest Version ==="
                    curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${subjectName}/versions/latest 2>/dev/null
                    echo
                    echo
                    echo "=== All Versions ==="
                    curl -s ${params.SCHEMA_REGISTRY_URL}/subjects/${subjectName}/versions 2>/dev/null
                    echo
                    echo
                    echo "=== Subject Compatibility ==="
                    curl -s ${params.SCHEMA_REGISTRY_URL}/config/${subjectName} 2>/dev/null || echo "Using global compatibility settings"
                ' 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        return latestVersionOutput
    } catch (Exception e) {
        return "ERROR: Failed to describe subject '${subjectName}' - ${e.getMessage()}"
    }
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
        textContent += "Available Schema Subjects:\n"
        textContent += "=" * 50 + "\n\n"
        
        subjects.eachWithIndex { subject, index ->
            textContent += "${index + 1}. ${subject}\n"
            
            if (params.INCLUDE_VERSIONS && subjectDetails.containsKey(subject)) {
                textContent += "   Versions: ${subjectDetails[subject]}\n"
            }
        }
        
        textContent += "\n" + "=" * 50 + "\n"
        textContent += "\n💡 To get detailed information about a specific subject, re-run with SUBJECT_NAME parameter.\n"
    }

    writeFile file: env.SCHEMA_SUBJECTS_FILE, text: textContent
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

    writeFile file: env.SCHEMA_SUBJECTS_FILE, text: textContent
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