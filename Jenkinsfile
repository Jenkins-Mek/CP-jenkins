//@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Enter topic name to describe (or leave empty to list available topics)')
    ])
])

pipeline {
    agent any

    environment {
        TOPICS_DESCRIBE_FILE = 'kafka-topics-describe.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Create Client Configuration') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('List Available Topics') {
            when {
                expression { !params.TOPIC_NAME?.trim() }
            }
            steps {
                script {
                    def topics = listKafkaTopics()
                    if (!topics || topics.isEmpty()) {
                        error("❌ No Kafka topics found.")
                    }

                    echo "📋 Available topics (${topics.size()}):"
                    topics.eachWithIndex { topic, i -> echo "  ${i + 1}. ${topic}" }

                    // Save to file for artifact
                    writeFile file: env.TOPICS_DESCRIBE_FILE, text: topics.join("\n")

                    echo "\n💡 Re-run this job with a TOPIC_NAME to describe a topic."
                }
            }
        }

        stage('Describe Kafka Topic') {
            when {
                expression { params.TOPIC_NAME?.trim() }
            }
            steps {
                script {
                    def inputTopic = params.TOPIC_NAME.trim()
                    def topics = listKafkaTopics()

                    if (!topics.contains(inputTopic)) {
                        def matches = topics.findAll { it.toLowerCase().contains(inputTopic.toLowerCase()) }
                        if (matches.size() == 1) {
                            echo "✅ Partial match found: '${matches[0]}'"
                            inputTopic = matches[0]
                        } else if (matches.size() > 1) {
                            echo "❌ Multiple matches found for '${params.TOPIC_NAME}':"
                            matches.each { echo "  - ${it}" }
                            error("Please provide a more specific topic name.")
                        } else {
                            echo "❌ Topic '${inputTopic}' not found."
                            error("Available topics:\n" + topics.take(10).collect { "  - $it" }.join('\n'))
                        }
                    } else {
                        echo "✅ Topic '${inputTopic}' found"
                    }

                    echo "📝 Describing topic: ${inputTopic}"
                    def description = describeKafkaTopic(inputTopic)
                    saveTopicDescriptionsToFile([(inputTopic): description])
                    echo "✅ Topic description saved to ${env.TOPICS_DESCRIBE_FILE}"
                }
            }
        }
    }

    post {
        always {
            script {
                cleanupClientConfig()
            }
        }
        success {
            script {
                archiveArtifacts artifacts: "${env.TOPICS_DESCRIBE_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "📦 Artifact '${env.TOPICS_DESCRIBE_FILE}' archived successfully."
            }
        }
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

def listKafkaTopics() {
    def topicsOutput = sh(
        script: """
            docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
            exec -T broker bash -c "
                export KAFKA_OPTS=''
                export JMX_PORT=''
                export KAFKA_JMX_OPTS=''
                unset JMX_PORT
                unset KAFKA_JMX_OPTS
                unset KAFKA_OPTS
                kafka-topics --list --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} --command-config ${env.CLIENT_CONFIG_FILE}
            " 2>/dev/null
        """,
        returnStdout: true
    ).trim()

    def allTopics = topicsOutput.split('\n').findAll { it.trim() != '' && !it.startsWith('WARNING') && !it.contains('FATAL') }
    return params.INCLUDE_INTERNAL ? allTopics : allTopics.findAll { !it.startsWith('_') }
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

def describeKafkaTopic(topicName) {
    try {
        def describeOutput = sh(
            script: """
                docker compose --project-directory ${params.COMPOSE_DIR} -f ${params.COMPOSE_DIR}/docker-compose.yml \\
                exec -T broker bash -c "
                    export KAFKA_OPTS=''
                    export JMX_PORT=''
                    export KAFKA_JMX_OPTS=''
                    unset JMX_PORT
                    unset KAFKA_JMX_OPTS
                    unset KAFKA_OPTS
                    kafka-topics --describe --topic ${topicName} --bootstrap-server ${params.KAFKA_BOOTSTRAP_SERVER} --command-config ${env.CLIENT_CONFIG_FILE}
                " 2>/dev/null
            """,
            returnStdout: true
        ).trim()

        return describeOutput
    } catch (Exception e) {
        return "ERROR: Failed to describe topic '${topicName}' - ${e.getMessage()}"
    }
}

def saveTopicDescriptionsToFile(topicDescriptions) {
    def timestamp = new Date().format('yyyy-MM-dd HH:mm:ss')
    def textContent = """# Kafka Topics Description
# Generated: ${timestamp}
# Total topics described: ${topicDescriptions.size()}
# Bootstrap server: ${params.KAFKA_BOOTSTRAP_SERVER}
# Security protocol: ${params.SECURITY_PROTOCOL}
# Specific topic: ${params.TOPIC_NAME ?: 'All topics'}

"""

    topicDescriptions.each { topicName, description ->
        textContent += """
================================================================================
Topic: ${topicName}
================================================================================
${description}

"""
    }

    writeFile file: env.TOPICS_DESCRIBE_FILE, text: textContent
}