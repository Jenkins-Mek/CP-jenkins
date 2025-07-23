@Library('kafka-ops-shared-lib') _

properties([
    parameters([
        string(name: 'COMPOSE_DIR', defaultValue: '/confluent/cp-mysetup/cp-all-in-one', description: 'Docker Compose directory path'),
        string(name: 'KAFKA_BOOTSTRAP_SERVER', defaultValue: 'localhost:9092', description: 'Kafka bootstrap server'),
        choice(name: 'SECURITY_PROTOCOL', choices: ['SASL_PLAINTEXT', 'SASL_SSL'], defaultValue: 'SASL_PLAINTEXT', description: 'Kafka security protocol'),
        // Active Choices parameter to read cached topics from a file on Jenkins master or shared storage
        [$class: 'CascadeChoiceParameter',
         choiceType: 'PT_SINGLE_SELECT',
         description: 'Select topic to describe',
         filterLength: 1,
         filterable: true,
         name: 'TOPIC_NAME',
         referencedParameters: '',
         script: [
            $class: 'GroovyScript',
            fallbackScript: [classpath: [], sandbox: true, script: 'return ["<error reading topics>"]'],
            script: [classpath: [], sandbox: true, script: '''
                def topicFilePath = "/var/jenkins_home/workspace/org-cp-tools_CP-jenkins_Refresh-Kafka-Topics-Cache/cached-kafka-topics.txt"
                def topics = []
                try {
                    File f = new File(topicFilePath)
                    if (f.exists()) {
                        topics = f.readLines()
                        if (topics.isEmpty()) {
                            topics = ["<no topics found>"]
                        }
                    } else {
                        topics = ["<topic cache file not found>"]
                    }
                } catch (Exception e) {
                    topics = ["<error: ${e.getMessage()}>"]
                }
                return topics
            ''']
         ]
        ]
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

        stage('Describe Kafka Topic') {
            steps {
                script {
                    if (!params.TOPIC_NAME?.trim() || params.TOPIC_NAME.startsWith("<")) {
                        error("Please select a valid topic from the dropdown.")
                    }
                    echo "Describing topic: ${params.TOPIC_NAME}"
                    def topicDescriptions = [:]
                    try {
                        def description = confluentOps.describeKafkaTopic(params.TOPIC_NAME.trim())
                        topicDescriptions[params.TOPIC_NAME.trim()] = description
                        confluentOps.saveTopicDescriptionsToFile(topicDescriptions)
                        echo "Topic description saved."
                    } catch (err) {
                        error("Failed to describe topic '${params.TOPIC_NAME}': ${err.getMessage()}")
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                archiveArtifacts artifacts: "${env.TOPICS_DESCRIBE_FILE}", fingerprint: true, allowEmptyArchive: true
                echo "Topic description archived."
            }
        }
        always {
            script {
                confluentOps.cleanupClientConfig()
            }
        }
    }
}
