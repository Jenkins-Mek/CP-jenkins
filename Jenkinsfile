properties([
    parameters([
        [$class: 'ChoiceParameter',
            choiceType: 'PT_SINGLE_SELECT',
            description: 'What topic operation do you want to perform?',
            filterLength: 1,
            filterable: false,
            name: 'OPERATION',
            script: [
                $class: 'GroovyScript',
                fallbackScript: [
                    classpath: [],
                    sandbox: true,
                    script:
                        '''return['CREATE_TOPIC:ERROR']'''
                ],
                script: [
                    classpath: [],
                    sandbox: true,
                    script:
                        '''return["CREATE_TOPIC","LIST_TOPICS:selected","DESCRIBE_TOPIC","DELETE_TOPIC"]'''
                ]
            ]
        ],
        [$class: 'DynamicReferenceParameter',
            choiceType: 'ET_FORMATTED_HTML',
            description: 'Topic Configuration Options',
            name: 'TOPIC_OPTIONS',
            omitValueField: false,
            referencedParameters: 'OPERATION',
            script: [
                $class: 'GroovyScript',
                fallbackScript: [
                    classpath: [],
                    sandbox: true,
                    script:
                        '''return['TOPIC_MANAGEMENT:ERROR']'''
                ],
                script: [
                    classpath: [],
                    sandbox: true,
                    script:
                        '''
                        if (OPERATION == 'LIST_TOPICS'){
                            return """
                                <div style="background-color: #e8f5e8; padding: 15px; border-radius: 5px; border-left: 4px solid #28a745;">
                                    <h4 style="margin: 0; color: #155724;">📋 List All Topics</h4>
                                    <p style="margin: 5px 0 0 0; color: #155724;">This operation will list all available Kafka topics with detailed information including count, names, partitions, and replication factors.</p>
                                    <div style="margin-top: 10px;">
                                        <label style="font-weight: bold; color: #155724;">
                                            <input type="checkbox" name="value" value="include_internal" style="margin-right: 5px;">
                                            Include internal topics (starting with _)
                                        </label>
                                    </div>
                                </div>
                            """
                        } else if (OPERATION == 'CREATE_TOPIC') {
                            return """
                                <div style="background-color: #f8f9fa; padding: 15px; border-radius: 5px; border: 1px solid #dee2e6;">
                                    <h4 style="margin: 0 0 15px 0; color: #495057;">🚀 Create New Topic</h4>
                                    <table style="width: 100%; border-collapse: collapse;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #495057;">Topic Name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='text' value='user-events' style="width: 300px; padding: 5px; border: 1px solid #ced4da; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #6c757d; margin-top: 3px;">Use alphanumeric characters, dots, underscores, and hyphens</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #495057;">Partitions *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #ced4da; border-radius: 3px;">
                                                    <option value='1' selected>1 (Development)</option>
                                                    <option value='3'>3 (Small workload)</option>
                                                    <option value='6'>6 (Medium workload)</option>
                                                    <option value='12'>12 (High workload)</option>
                                                    <option value='24'>24 (Very high workload)</option>
                                                </select>
                                                <div style="font-size: 12px; color: #6c757d; margin-top: 3px;">More partitions = better parallelism but more overhead</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #495057;">Replication Factor *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #ced4da; border-radius: 3px;">
                                                    <option value='1' selected>1 (Development - No redundancy)</option>
                                                    <option value='2'>2 (Staging - Basic redundancy)</option>
                                                    <option value='3'>3 (Production - High availability)</option>
                                                </select>
                                                <div style="font-size: 12px; color: #6c757d; margin-top: 3px;">Production should use 3 for fault tolerance</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'DESCRIBE_TOPIC') {
                            return """
                                <div style="background-color: #fff3cd; padding: 15px; border-radius: 5px; border-left: 4px solid #ffc107;">
                                    <h4 style="margin: 0 0 15px 0; color: #856404;">🔍 Describe Topic</h4>
                                    <table style="width: 100%;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #856404;">Topic Name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='text' value='user-events' style="width: 300px; padding: 5px; border: 1px solid #ffeaa7; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Enter the name of an existing topic to get its details</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'DELETE_TOPIC') {
                            return """
                                <div style="background-color: #f8d7da; padding: 15px; border-radius: 5px; border-left: 4px solid #dc3545;">
                                    <h4 style="margin: 0 0 15px 0; color: #721c24;">⚠️ Delete Topic</h4>
                                    <div style="background-color: #ffffff; padding: 10px; border-radius: 3px; margin-bottom: 15px; border: 1px solid #f5c6cb;">
                                        <strong style="color: #721c24;">⚠️ WARNING:</strong> This action will permanently delete the topic and all its data. This cannot be undone!
                                    </div>
                                    <table style="width: 100%;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #721c24;">Topic Name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='text' value='' placeholder='Enter topic name to delete' style="width: 300px; padding: 5px; border: 1px solid #f5c6cb; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #721c24; margin-top: 3px;">You will be asked to confirm the deletion before it proceeds</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else {
                            return """
                                <div style="background-color: #d1ecf1; padding: 15px; border-radius: 5px; border-left: 4px solid #17a2b8;">
                                    <h4 style="margin: 0; color: #0c5460;">Select an Operation</h4>
                                    <p style="margin: 5px 0 0 0; color: #0c5460;">Please choose a topic operation from the dropdown above.</p>
                                </div>
                            """
                        }
                        '''
                ]
            ]
        ]
    ])
])

pipeline {
    agent any
    
    environment {
        COMPOSE_DIR = '/confluent/cp-mysetup/cp-all-in-one'
        CONNECTION_TYPE = 'local-confluent'
        KAFKA_BOOTSTRAP_SERVER = 'localhost:9092'
        SECURITY_PROTOCOL = 'SASL_PLAINTEXT'
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        CLIENT_CONFIG_FILE = '/tmp/client.properties'
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "🚀 Starting Kafka Topic Management"
                    echo "Operation: ${params.OPERATION}"
                }
            }
        }

        stage('Parse Parameters') {
            steps {
                script {
                    def option = "${params.TOPIC_OPTIONS}"
                    def values = option.split(',').collect { it.trim() }.findAll { it }

                    switch(params.OPERATION) {
                        case 'CREATE_TOPIC':
                            env.TOPIC_NAME = values[0]
                            env.PARTITIONS = values[1] ?: '3'
                            env.REPLICATION_FACTOR = values[2] ?: '2'
                            echo "Creating topic: ${env.TOPIC_NAME} (${env.PARTITIONS} partitions, ${env.REPLICATION_FACTOR} replicas)"
                            break
                        case 'DESCRIBE_TOPIC':
                        case 'DELETE_TOPIC':
                            env.TOPIC_NAME = values[0]
                            echo "Topic: ${env.TOPIC_NAME}"
                            break
                        case 'LIST_TOPICS':
                            env.INCLUDE_INTERNAL = values.contains('include_internal') ? 'true' : 'false'
                            echo "Listing all topics (Include internal: ${env.INCLUDE_INTERNAL})"
                            break
                    }
                }
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'CREATE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            if (!env.TOPIC_NAME.matches('^[a-zA-Z0-9._-]+$')) error "Invalid topic name format"
                            echo "✅ Validation passed"
                            break
                        case 'DESCRIBE_TOPIC':
                        case 'DELETE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            echo "✅ Validation passed"
                            break
                        case 'LIST_TOPICS':
                            echo "✅ Validation passed"
                            break
                    }
                }
            }
        }

        stage('Create Client Configuration') {
            when {
                expression { params.OPERATION == 'LIST_TOPICS' }
            }
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

        stage('Execute Operation') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'LIST_TOPICS':
                            echo "📋 Listing Kafka topics..."
                            def topics = listKafkaTopics()
                            
                            if (topics.size() > 0) {
                                echo "✅ Found ${topics.size()} topic(s):"
                                topics.eachWithIndex { topic, index ->
                                    echo "  ${index + 1}. ${topic}"
                                }
                                saveTopicsToFile(topics)
                                echo "💾 Topics saved to ${env.TOPICS_LIST_FILE}"
                            } else {
                                echo "⚠️ No topics found"
                                writeFile file: env.TOPICS_LIST_FILE, text: "# No topics found\n"
                            }
                            break

                        case 'CREATE_TOPIC':
                            echo "==== Calling Create Topic job ===="
                            build job: 'GIT-org/jenkins1/create-topic',
                                parameters: [
                                    string(name: 'TopicName', value: "${env.TOPIC_NAME}"),
                                    string(name: 'Partitions', value: "${env.PARTITIONS}"),
                                    string(name: 'ReplicationFactor', value: "${env.REPLICATION_FACTOR}"),
                                    string(name: 'ParamsAsENV', value: 'true'),
                                    string(name: 'ENVIRONMENT_PARAMS', value: "${env.COMPOSE_DIR},${env.CONNECTION_TYPE}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DESCRIBE_TOPIC':
                            echo "==== Calling Describe Topic job ===="
                            build job: 'GIT-org/jenkins1/describe-topic',
                                parameters: [
                                    string(name: 'TopicName', value: "${env.TOPIC_NAME}"),
                                    string(name: 'ParamsAsENV', value: 'true'),
                                    string(name: 'ENVIRONMENT_PARAMS', value: "${env.COMPOSE_DIR},${env.CONNECTION_TYPE}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DELETE_TOPIC':
                            echo "⚠️ Requesting delete confirmation..."
                            def confirmName = input(
                                message: "Delete topic '${env.TOPIC_NAME}'? This cannot be undone!",
                                parameters: [string(name: 'CONFIRM_NAME', description: "Type topic name to confirm")]
                            )
                            if (confirmName != env.TOPIC_NAME) {
                                error "❌ Confirmation failed - typed '${confirmName}' but expected '${env.TOPIC_NAME}'"
                            }
                            echo "✅ Confirmation successful, proceeding with deletion..."

                            echo "==== Calling Delete Topic job ===="
                            build job: 'GIT-org/jenkins1/delete-topic', 
                                parameters: [
                                    string(name: 'TopicName', value: "${env.TOPIC_NAME}"),
                                    string(name: 'ParamsAsENV', value: 'true'),
                                    string(name: 'ENVIRONMENT_PARAMS', value: "${env.COMPOSE_DIR},${env.CONNECTION_TYPE}")
                                ],
                                propagate: false,
                                wait: true
                            break

                        default:
                            error "❌ Unknown operation: ${params.OPERATION}"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                if (params.OPERATION == 'LIST_TOPICS') {
                    archiveArtifacts artifacts: "${env.TOPICS_LIST_FILE}",
                                   fingerprint: true,
                                   allowEmptyArchive: true
                    echo "📦 Topics list archived as artifact"
                }
                echo "✅ Kafka topic operation '${params.OPERATION}' completed successfully"
            }
        }
        failure {
            echo "❌ Kafka topic operation '${params.OPERATION}' failed - check logs for details"
        }
        always {
            script {
                if (params.OPERATION == 'LIST_TOPICS') {
                    cleanupClientConfig()
                }
                echo "🧹 Cleaning up temporary environment variables"
            }
        }
    }
}
