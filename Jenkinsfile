properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        booleanParam(name: 'INCLUDE_INTERNAL', defaultValue: false, description: 'Include internal Kafka topics (starting with _)'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['PLAINTEXT', 'SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol')
    ])
])

pipeline {
    agent any

    environment {
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Create Client Configuration') {
            steps {
                script {
                    echo "🔧 Creating Kafka client configuration..."
                    withCredentials([
                        usernamePassword(
                            credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9',
                            usernameVariable: 'KAFKA_USERNAME',
                            passwordVariable: 'KAFKA_PASSWORD'
                        )
                    ]) {
                        createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                    echo "✅ Client configuration created"
                }
            }
        }

        stage('List Kafka Topics') {
            steps {
                script {
                    echo "📋 Retrieving Kafka topics..."
                    def topics = listKafkaTopics()
                    
                    if (topics.size() > 0) {
                        echo "✅ Found ${topics.size()} topic(s)"
                        topics.eachWithIndex { topic, index ->
                            echo "  ${index + 1}. ${topic}"
                        }
                        saveTopicsToFile(topics)
                    } else {
                        echo "⚠️ No topics found"
                        writeFile file: env.TOPICS_LIST_FILE, text: "# No topics found\n"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                archiveArtifacts artifacts: "${env.TOPICS_LIST_FILE}",
                               fingerprint: true,
                               allowEmptyArchive: true
                echo "📦 Topics list archived"
            }
        }
        failure {
            echo "❌ Failed to list topics - check Kafka services and configuration"
        }
        always {
            script {
                cleanupClientConfig()
            }
        }
    }
}

// ========== HELPER FUNCTIONS ==========

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
        case 'PLAINTEXT':
        default:
            securityConfig = """
security.protocol=PLAINTEXT
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

def listKafkaTopics() {
    def topicsOutput = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "
                kafka-topics --list --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} --command-config ${env.CLIENT_CONFIG_FILE}
            " 2>/dev/null
        """,
        returnStdout: true
    ).trim()

    def allTopics = topicsOutput.split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') }
    return params.INCLUDE_INTERNAL ? allTopics : allTopics.findAll { !it.startsWith('_') }
}

def saveTopicsToFile(topics) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def textContent = """# Kafka Topics List
# Generated: ${timestamp}
# Total topics: ${topics.size()}
# Include internal: ${params.INCLUDE_INTERNAL}
# Bootstrap server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security protocol: ${params.SECURITY_PROTOCOL}

"""
    topics.each { topic ->
        textContent += "${topic}\n"
    }
    
    writeFile file: env.TOPICS_LIST_FILE, text: textContent
}

def cleanupClientConfig() {
    try {
        sh """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "rm -f ${env.CLIENT_CONFIG_FILE}" 2>/dev/null || true
        """
    } catch (Exception e) {
        // Ignore cleanup errors
    }
}