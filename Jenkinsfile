@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        string(name: 'TOPIC_NAME', defaultValue: '', description: 'Topic name to describe (leave empty to describe all topics)'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol')
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
                    echo "🔧 Creating Kafka client configuration..."
                    withCredentials([
                        usernamePassword(
                            credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9',
                            usernameVariable: 'KAFKA_USERNAME',
                            passwordVariable: 'KAFKA_PASSWORD'
                        )
                    ]) {
                        confluentOps.createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
                    }
                    echo "✅ Client configuration created"
                }
            }
        }

        stage('Describe Kafka Topics') {
            steps {
                script {
                    echo "📋 Describing Kafka topics..."

                    def topicsToDescribe = []

                    if (params.TOPIC_NAME?.trim()) {
                        // Describe specific topic
                        topicsToDescribe = [params.TOPIC_NAME.trim()]
                        echo "🎯 Describing specific topic: ${params.TOPIC_NAME}"
                    }

                    if (topicsToDescribe.size() > 0) {
                        def topicDescriptions = [:]

                        topicsToDescribe.each { topic ->
                            echo "🔍 Describing topic: ${topic}"
                            def description = confluentOps.describeKafkaTopic(topic)
                            topicDescriptions[topic] = description
                        }

                        confluentOps.saveTopicDescriptionsToFile(topicDescriptions)
                        echo "✅ Successfully described ${topicsToDescribe.size()} topic(s)"
                    } else {
                        echo "⚠️ No topics found to describe"
                        writeFile file: env.TOPICS_DESCRIBE_FILE, text: "# No topics found to describe\n"
                    }
                }
            }
        }stage('Describe Kafka Topics') {
            steps {
                script {
                    echo "📋 Describing Kafka topics..."

                    def topicsToDescribe = []

                    if (params.TOPIC_NAME?.trim()) {
                        topicsToDescribe = [params.TOPIC_NAME.trim()]
                        echo "🎯 Attempting to describe specific topic: ${params.TOPIC_NAME}"
                    }

                    if (topicsToDescribe.size() > 0) {
                        def topicDescriptions = [:]
                        def failedTopics = []

                        topicsToDescribe.each { topic ->
                            try {
                                echo "🔍 Describing topic: ${topic}"
                                def description = confluentOps.describeKafkaTopic(topic)
                                topicDescriptions[topic] = description
                            } catch (err) {
                            echo "⚠️ Failed to describe topic '${topic}': ${err.getMessage()}"
                            failedTopics << topic
                            }
                        }

                    if (!topicDescriptions.isEmpty()) {
                        confluentOps.saveTopicDescriptionsToFile(topicDescriptions)
                        echo "✅ Successfully described ${topicDescriptions.size()} topic(s)"
                    }


                    if (!failedTopics.isEmpty()) {
                        echo "📄 Listing all topics as fallback for failed description(s): ${failedTopics}"
                        def topicList = confluentOps.listKafkaTopics()
                        def fallbackText = "# Failed to describe: ${failedTopics.join(', ')}\n# Available topics:\n"
                        fallbackText += topicList.join("\n")
                        writeFile file: env.TOPICS_DESCRIBE_FILE, text: fallbackText
                        echo "📋 Fallback topic list written to file"
                    }
                    } else {
                        echo "⚠️ No topic name provided"
                        writeFile file: env.TOPICS_DESCRIBE_FILE, text: "# No topic name provided\n"
                    }
                }
            }
        }

    }

    post {
        success {
            script {
                archiveArtifacts artifacts: "${env.TOPICS_DESCRIBE_FILE}",
                               fingerprint: true,
                               allowEmptyArchive: true
                echo "📦 Topic descriptions archived"
            }
        }
        failure {
            echo "❌ Failed to describe topics - check Kafka services and configuration"
        }
        always {
            script {
                confluentOps.cleanupClientConfig()
            }
        }
    }
}