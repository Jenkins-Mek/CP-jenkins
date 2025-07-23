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
    }

    stages {
        stage('Create Client Configuration') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '2cc1527f-e57f-44d6-94e9-7ebc53af65a9', usernameVariable: 'KAFKA_USERNAME', passwordVariable: 'KAFKA_PASSWORD')]) {
                        confluentOps.createKafkaClientConfig(env.KAFKA_USERNAME, env.KAFKA_PASSWORD)
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
                    def topics = confluentOps.listKafkaTopics()
                    if (!topics || topics.isEmpty()) {
                        error("❌ No Kafka topics found.")
                    }
                    echo "📋 Available topics (${topics.size()}):"
                    topics.eachWithIndex { topic, i -> echo "  ${i + 1}. ${topic}" }
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
                    def topics = confluentOps.listKafkaTopics()

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
                    def description = confluentOps.describeKafkaTopic(inputTopic)
                    confluentOps.saveTopicDescriptionsToFile([(inputTopic): description])
                    echo "✅ Topic description saved to ${env.TOPICS_DESCRIBE_FILE}"
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.TOPIC_NAME?.trim()) {
                    archiveArtifacts artifacts: "${env.TOPICS_DESCRIBE_FILE}", fingerprint: true, allowEmptyArchive: true
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
