properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'SCHEMA_REGISTRY_URL', defaultValue: 'http://localhost:8081', description: 'Schema Registry URL'),
        string(name: 'SUBJECT_NAME', defaultValue: '', description: 'Subject name to delete (e.g., user-topic-value)')
    ])
])

pipeline {
    agent any

    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.SUBJECT_NAME?.trim()) {
                        error("❌ SUBJECT_NAME is required.")
                    }
                    echo "📋 Preparing to delete subject: ${params.SUBJECT_NAME}"
                }
            }
        }

        stage('Delete Schema') {
            steps {
                script {
                    deleteSchemaSubject()
                }
            }
        }
    }

    post {
        success {
            echo "✅ Schema subject '${params.SUBJECT_NAME}' deleted successfully."
        }
        failure {
            echo "❌ Failed to delete schema subject '${params.SUBJECT_NAME}'."
        }
    }
}

def deleteSchemaSubject() {
    def response = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T schema-registry bash -c '
                curl -s -w "\\n%{http_code}" -X DELETE \\
                ${params.SCHEMA_REGISTRY_URL}/subjects/${params.SUBJECT_NAME}
            '
        """,
        returnStdout: true
    ).trim()

    def lines = response.split('\\n')
    def httpCode = lines[-1]
    def responseBody = lines.size() > 1 ? lines[0..-2].join('\\n') : ''

    echo "🗑️ Delete response: ${responseBody}"

    if (httpCode.startsWith('2')) {
        echo "✅ Schema deleted (HTTP ${httpCode})"
    } else {
        error("❌ Schema delete failed (HTTP ${httpCode}): ${responseBody}")
    }
}
