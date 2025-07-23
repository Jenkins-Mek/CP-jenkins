@Library('kafka-ops-shared-lib') _

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
        CACHED_TOPICS_FILE = 'cached-kafka-topics.txt'
    }

    stages {
        stage('Copy Topic Cache') {
            steps {
                script {
                    try {
                        // Copy the cached topics file from the previous job
                        copyArtifacts(
                            projectName: 'org-cp-tools/CP-jenkins/Refresh-Kafka-Topics-Cache',
                            filter: 'cached-kafka-topics.txt',
                            target: '.',
                            selector: lastSuccessful()
                        )
                        echo "✅ Successfully copied cached topics file"
                    } catch (Exception e) {
                        error("❌ Failed to copy cached topics file. Please run 'Refresh-Kafka-Topics-Cache' job first. Error: ${e.getMessage()}")
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
                    if (fileExists(env.CACHED_TOPICS_FILE)) {
                        def topics = readFile(env.CACHED_TOPICS_FILE).split('\n')
                        echo "📋 Available topics (${topics.size()}):"
                        topics.eachWithIndex { topic, index ->
                            if (topic.trim()) {
                                echo "  ${index + 1}. ${topic.trim()}"
                            }
                        }
                        echo "\n💡 Re-run this job with a TOPIC_NAME parameter to describe a specific topic."
                    } else {
                        error("❌ Cached topics file not found")
                    }
                }
            }
        }

        stage('Create Client Configuration') {
            when {
                expression { params.TOPIC_NAME?.trim() }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        confluentOps.createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                }
            }
        }

        stage('Validate Topic Selection') {
            when {
                expression { params.TOPIC_NAME?.trim() }
            }
            steps {
                script {
                    def topics = readFile(env.CACHED_TOPICS_FILE).split('\n').collect { it.trim() }.findAll { it }
                    def selectedTopic = params.TOPIC_NAME.trim()
                    
                    if (!topics.contains(selectedTopic)) {
                        echo "❌ Topic '${selectedTopic}' not found in cache."
                        echo "📋 Available topics:"
                        topics.each { topic -> echo "  - ${topic}" }
                        error("Please select a valid topic from the list above.")
                    } else {
                        echo "✅ Topic '${selectedTopic}' found in cache. Proceeding with description."
                    }
                }
            }
        }

        stage('Describe Kafka Topic') {
            when {
                expression { params.TOPIC_NAME?.trim() }
            }
            steps {
                script {
                    echo "📝 Describing topic: ${params.TOPIC_NAME}"
                    def topicDescriptions = [:]
                    try {
                        def description = confluentOps.describeKafkaTopic(params.TOPIC_NAME.trim())
                        topicDescriptions[params.TOPIC_NAME.trim()] = description
                        confluentOps.saveTopicDescriptionsToFile(topicDescriptions)
                        echo "✅ Topic description saved successfully."
                    } catch (err) {
                        error("❌ Failed to describe topic '${params.TOPIC_NAME}': ${err.getMessage()}")
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.TOPIC_NAME?.trim()) {
                    archiveArtifacts artifacts: "${env.TOPICS_DESCRIBE_FILE}", fingerprint: true, allowEmptyArchive: true
                    echo "📦 Topic description archived successfully."
                }
            }
        }
        always {
            script {
                confluentOps.cleanupClientConfig()
            }
        }
    }
}