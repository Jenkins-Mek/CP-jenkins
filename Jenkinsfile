@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        [$class: 'CascadeChoiceParameter',
            choiceType: 'PT_SINGLE_SELECT',
            description: 'Select topic to describe (leave blank to list all)',
            filterLength: 1,
            filterable: true,
            name: 'TOPIC_NAME',
            referencedParameters: 'KAFKA_BOOTSTRAP_SERVER,SECURITY_PROTOCOL',
            script: [
                $class: 'GroovyScript',
                fallbackScript: [classpath: [], sandbox: true, script: 'return ["<error listing topics>"]'],
                script: [classpath: [], sandbox: true, script: '''
                    def kafka = com.mycompany.ConfluentOps.getInstance()
                    def topics = kafka.listKafkaTopics()
                    return topics ?: ["<no topics found>"]
                ''']
            ]
        ],
        booleanParam(name: 'INCLUDE_INTERNAL', defaultValue: false, description: 'Include internal topics')
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

                    if (!params.TOPIC_NAME?.trim()) {
                        echo "⚠️ No topic name provided. Listing all available topics for manual selection..."

                        def allTopics = confluentOps.listKafkaTopics()
                        def listText = "# No topic selected.\n# Available topics:\n" + allTopics.join("\n")
                        writeFile file: env.TOPICS_DESCRIBE_FILE, text: listText
                        error("Please re-run the job and select a topic from the dropdown list.")
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