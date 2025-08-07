
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
                        '''return ['ERROR: Unable to load operations']'''
                ],
                script: [
                    classpath: [],
                    sandbox: true,
                    script: '''
                        return [
                            "LIST_TOPICS:selected",
                            "CREATE_TOPIC",
                            "DESCRIBE_TOPIC",
                            "ALTER_TOPIC",
                            "DELETE_TOPIC",
                            "PRODUCER",
                            "CONSUMER",
                            "LIST_SCHEMA",
                            "REGISTER_SCHEMA",
                            "DESCRIBE_SCHEMA",
                            "DELETE_SCHEMA"
                        ]
                    '''
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
                        def readHtmlFromFile(String operation) {
                            def htmlFile = new File('/var/lib/jenkins/workspace/html-store.txt')
                            def lines = htmlFile.readLines()
                            def htmlContent = ""
                            def isInSection = false
                            lines.each { line ->
                                if (line.trim().startsWith(operation + " =")) {
                                    isInSection = true
                                } else if (line.trim().contains(" =") && isInSection) {
                                    isInSection = false
                                } else if (isInSection) {
                                    htmlContent += line + "\\n"
                                }
                            }
                            return htmlContent.trim() ?: " "
                        }

                        def getTopics() {
                            def topics = []
                            try {
                                def filePath = '/var/lib/jenkins/workspace/kafka-topics-list.txt'
                                def choicesFile = new File(filePath)
                                if (choicesFile.exists()) {
                                    topics = choicesFile.readLines()
                                        .collect { it.trim() }
                                        .findAll { it && !it.startsWith('#') }
                                        .sort()
                                } else {
                                    topics = ["ERROR: File not found: ${filePath}"]
                                }
                            } catch (Exception e) {
                                topics = ["ERROR: ${e.message}"]
                            }
                            return topics
                        }

                        def getSchemaSubjects() {
                            def subjects = []
                                try {
                                    def filePath = '/var/lib/jenkins/workspace/schema-subjects-list.txt'
                                    def choicesFile = new File(filePath)
                                    if (choicesFile.exists()) {
                                        choicesFile.readLines()
                                            .collect { it.trim() }
                                            .findAll { it && !it.startsWith('#') }
                                            .each { line ->
                                                // Parse format: subject-name[version1,version2,...]
                                                if (line.contains('[') && line.endsWith(']')) {
                                                    def subjectName = line.substring(0, line.indexOf('['))
                                                    subjects << subjectName
                                                } else {
                                                    subjects << line
                                                }
                                            }
                                        subjects = subjects.sort()
                                    } else {
                                        subjects = ["ERROR: File not found: ${filePath}"]
                                    }
                                } catch (Exception e) {
                                    subjects = ["ERROR: ${e.message}"]
                                }
                                return subjects
                        }

                        // Main logic
                        if (OPERATION == 'LIST_TOPICS'){
                            return readHtmlFromFile('LIST_TOPICS')
                        }else if (OPERATION == 'CREATE_TOPIC') {
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
                        } else if (OPERATION == 'ALTER_TOPIC') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">'
                            topicOptions += '<option value="">-- Select Topic --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            return """
                                <div style="background-color: #fff3cd; padding: 15px; border-radius: 5px; border: 1px solid #ffeeba;">
                                    <h4 style="margin: 0 0 15px 0; color: #856404;">⚙️ Alter Topic Configuration</h4>
                                    <table style="width: 100%; border-collapse: collapse;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #856404;">Select Topic *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Select the topic to modify from the dropdown</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #856404;">Retention Days</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='number' value='7' min='1' style="width: 150px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">How many days messages are retained in the topic</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #856404;">Cleanup Policy</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">
                                                    <option value='delete' selected>delete</option>
                                                    <option value='compact'>compact</option>
                                                    <option value='delete|compact'>delete,compact</option>
                                                </select>
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Topic cleanup policy</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #856404;">Segment Bytes</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='number' value='1073741824' placeholder="e.g. 1073741824" style="width: 200px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Segment size in bytes (default: 1GB)</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #856404;">Min In-Sync Replicas</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='number' value='1' min='1' style="width: 150px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Minimum in-sync replicas required</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #856404;">Max Message Bytes</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name='value' type='number' value='1000000' placeholder="e.g. 1000000" style="width: 200px; padding: 5px; border: 1px solid #ffe8a1; border-radius: 3px;">
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Maximum message size in bytes (default: 1MB)</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'DESCRIBE_TOPIC') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #ffeaa7; border-radius: 3px;">'
                            topicOptions += '<option value="">-- Select Topic --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            return """
                                <div style="background-color: #fff3cd; padding: 15px; border-radius: 5px; border-left: 4px solid #ffc107;">
                                    <h4 style="margin: 0 0 15px 0; color: #856404;">🔍 Describe Topic</h4>
                                    <table style="width: 100%;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #856404;">Select Topic *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 12px; color: #856404; margin-top: 3px;">Select an existing topic to get its details</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'DELETE_TOPIC') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 2px solid #dc3545; border-radius: 3px; background-color: #fff2f2;">'
                            topicOptions += '<option value="">-- Select Topic to Delete --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            return """
                                <div style="background-color: #f8d7da; padding: 15px; border-radius: 5px; border-left: 4px solid #dc3545;">
                                    <h4 style="margin: 0 0 15px 0; color: #721c24;">⚠️ Delete Topic</h4>
                                    <div style="background-color: #ffffff; padding: 10px; border-radius: 3px; margin-bottom: 15px; border: 1px solid #f5c6cb;">
                                        <strong style="color: #721c24;">⚠️ WARNING:</strong> This action will permanently delete the topic and all its data. This cannot be undone!
                                    </div>
                                    <table style="width: 100%;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #721c24;">Select Topic to Delete *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 12px; color: #721c24; margin-top: 3px;">⚠️ Carefully select the topic you want to permanently delete</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        }else if (OPERATION == 'PRODUCER') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 2px solid #c3e6cb; border-radius: 3px; background-color: #ecf6eeff;">'
                            topicOptions += '<option value="">-- Select Topic to Produce --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            def subjectOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #c3e6cb; border-radius: 3px; background-color: #ecf6eeff">'
                            subjectOptions += '<option value="">-- Select Schema Subject to Produce --</option>'
                            getSchemaSubjects().each { subject ->
                               subjectOptions += "<option value='${subject}'>${subject}</option>"
                            }
                            subjectOptions += '</select>'

                            return """
                                <div style="background: linear-gradient(135deg, #d4edda 0%, #bbdefb 100%); padding: 20px; border-radius: 12px; border-left: 4px solid #28a745; margin: 10px 0; box-shadow: 0 4px 12px rgba(33,150,243,0.15);">
                                    <h4 style="margin: 0 0 18px 0; color: #155724; display: flex; align-items: center; font-size: 18px;">
                                        <span style="margin-right: 10px; font-size: 24px;">📤</span>
                                        Kafka Producer
                                    </h4>
                                    <table style="width: 100%;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #155724;">Producer Type *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name="value" style="width: 200px; padding: 5px; border: 1px solid #c3e6cb; border-radius: 3px;" onchange="toggleProducerFields(this.value)">
                                                    <option value="standard" selected>Standard Producer</option>
                                                    <option value="schema">Schema-based Producer</option>
                                                </select>
                                                <div style="font-size: 12px; color: #155724; margin-top: 3px;">Choose producer type</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #155724;">Topic Name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 12px; color: #155724; margin-top: 3px;">Select topic to produce to</div>
                                            </td>
                                        </tr>
                                        <tr class="schema-fields" style="display: none;">
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #155724;">Schema Subject *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${subjectOptions}
                                                <div style="font-size: 12px; color: #155724; margin-top: 3px;">Schema registry subject</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #155724;">Message Count</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name="value" style="width: 200px; padding: 5px; border: 1px solid #c3e6cb; border-radius: 3px;">
                                                    <option value="1" selected>1 message</option>
                                                    <option value="5">5 messages</option>
                                                    <option value="10">10 messages</option>
                                                    <option value="50">50 messages</option>
                                                </select>
                                                <div style="font-size: 12px; color: #155724; margin-top: 3px;">Number of messages to produce</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #455724;">Messages to produce *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <textarea name='value' type='text' value='' style="width: 350px; height: 100px; padding: 5px; border: 1px solid #c3e6cb; border-radius: 3px;"></textarea>
                                                <div style="font-size: 12px; color: #155724; margin-top: 3px;">Message</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'CONSUMER') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid  #b3d7ff; border-radius: 3px;">'
                            topicOptions += '<option value="">-- Select Topic --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            return """
                                <div style="background: linear-gradient(135deg, #e3f2fd 0%, #bbdefb 100%); padding: 20px; border-radius: 12px; border-left: 5px solid #2196f3; margin: 10px 0; box-shadow: 0 4px 12px rgba(33,150,243,0.15);">
                                    <h4 style="margin: 0 0 18px 0; color: #0d47a1; display: flex; align-items: center; font-size: 18px;">
                                        <span style="margin-right: 10px; font-size: 24px;">📥</span>
                                        Kafka Consumer Configuration
                                    </h4>

                                    <table style="width: 100%; border-collapse: collapse;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 180px;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Topic Name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Select the Kafka topic to consume from</div>
                                            </td>
                                        </tr>

                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Consumer Group</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name="value" type="text" placeholder="Enter consumer group ID (optional)" style="width: 300px; padding: 8px; border: 1px solid #b3d7ff; border-radius: 5px; background: white;">
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Consumer group ID for offset management</div>
                                            </td>
                                        </tr>

                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Offset Reset</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name="value" style="width: 200px; padding: 8px; border: 1px solid #b3d7ff; border-radius: 5px; background: white;">
                                                    <option value="latest" selected>Latest</option>
                                                    <option value="earliest">Earliest</option>
                                                </select>
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Starting position for consumption</div>
                                            </td>
                                        </tr>

                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Max Messages</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name="value" type="number" value="10" min="1" max="1000" style="width: 150px; padding: 8px; border: 1px solid #b3d7ff; border-radius: 5px; background: white;">
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Maximum number of messages to consume</div>
                                            </td>
                                        </tr>

                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Timeout (seconds)</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <input name="value" type="number" value="30" min="5" max="300" style="width: 150px; padding: 8px; border: 1px solid #b3d7ff; border-radius: 5px; background: white;">
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Consumer timeout duration</div>
                                            </td>
                                        </tr>

                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: 600; color: #1565c0; font-size: 14px;">Message Format</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name="value" style="width: 200px; padding: 8px; border: 1px solid #b3d7ff; border-radius: 5px; background: white;" onchange="toggleSchemaField(this.value)">
                                                    <option value="JSON" selected>JSON</option>
                                                    <option value="AVRO">AVRO</option>
                                                    <option value="JSON_SCHEMA">JSON Schema</option>
                                                    <option value="PROTOBUF">Protobuf</option>
                                                </select>
                                                <div style="font-size: 11px; color: #1976d2; margin-top: 3px;">Format of messages in the topic</div>
                                            </td>
                                        </tr>

                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'REGISTER_SCHEMA') {

                            def topicOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #dda0dd; border-radius: 3px;">'
                            topicOptions += '<option value="">-- Select Topic --</option>'
                            getTopics().each { topic ->
                                topicOptions += "<option value='${topic}'>${topic}</option>"
                            }
                            topicOptions += '</select>'

                            return """
                                <div style="background-color: #f8f0ff; padding: 15px; border-radius: 5px; border-left: 4px solid #8a2be2;">
                                    <h4 style="margin: 0 0 15px 0; color: #4b0082;">📋➕ Register New Schema</h4>
                                    <table style="width: 100%; border-collapse: collapse;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                <label style="font-weight: bold; color: #4b0082;">Topic name *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                ${topicOptions}
                                                <div style="font-size: 12px; color: #4b0082; margin-top: 3px;">Topic name for registered</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #4b0082;">Schema For</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #dda0dd; border-radius: 3px;">
                                                    <option value='key' >Key</option>
                                                    <option value='value'selected>Value</option>
                                                </select>
                                                <div style="font-size: 12px; color: #4b0082; margin-top: 3px;">Schema format type</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #4b0082;">Schema Type</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #dda0dd; border-radius: 3px;">
                                                    <option value='AVRO' selected>AVRO</option>
                                                    <option value='JSON'>JSON Schema</option>
                                                    <option value='PROTOBUF'>Protocol Buffers</option>
                                                </select>
                                                <div style="font-size: 12px; color: #4b0082; margin-top: 3px;">Schema format type</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #4b0082;">Compatibility Mode</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <select name='value' style="width: 200px; padding: 5px; border: 1px solid #dda0dd; border-radius: 3px;">
                                                    <option value='BACKWARD' selected>Backward</option>
                                                    <option value='FORWARD'>Forward</option>
                                                    <option value='FULL'>Full</option>
                                                    <option value='NONE'>None</option>
                                                </select>
                                                <div style="font-size: 12px; color: #4b0082; margin-top: 3px;">Schema evolution compatibility</div>
                                            </td>
                                        </tr>
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #4b0082;">Schema Content *</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <textarea name='value' type='text' value='' style="width: 300px; height: 400px; padding: 5px; border: 1px solid #dda0dd; border-radius: 3px;"></textarea>
                                                <div style="font-size: 12px; color: #4b0082; margin-top: 3px;">Schema definition</div>
                                            </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'DELETE_SCHEMA') {

                                def subjectOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #e9dfdfff; border-radius: 3px; background-color: #fff2f2;">'
                                subjectOptions += '<option value="">-- Select Schema Subject to Delete --</option>'
                                getSchemaSubjects().each { subject ->
                                   subjectOptions += "<option value='${subject}'>${subject}</option>"
                                }
                                subjectOptions += '</select>'

                                return """
                                   <div style="background-color: #ffe6e6; padding: 15px; border-radius: 5px; border-left: 4px solid #ff4444;">
                                       <h4 style="margin: 0 0 15px 0; color: #cc0000;">📋🗑️ Delete Schema - Step 1: Select Subject</h4>
                                        <div style="background-color: #ffffff; padding: 10px; border-radius: 3px; margin-bottom: 15px; border: 1px solid #ffcccc;">
                                           <strong style="color: #cc0000;">⚠️ WARNING:</strong> Deleting a schema can break existing producers and consumers. Ensure no active applications are using this schema.
                                       </div>
                                       <table style="width: 100%; border-collapse: collapse;">
                                          <tr>
                                               <td style="padding: 8px; vertical-align: top; width: 200px;">
                                                   <label style="font-weight: bold; color: #cc0000;">Subject Name *</label>
                                               </td>
                                               <td style="padding: 8px;">
                                                   ${subjectOptions}
                                                  <div style="font-size: 12px; color: #cc0000; margin-top: 3px;">⚠️ Select the schema subject</div>
                                               </td>
                                           </tr>
                                       </table>
                                   </div>
                                """
                        } else if (OPERATION == 'DESCRIBE_SCHEMA') {

                                def subjectOptions = '<select name="value" style="width: 300px; padding: 5px; border: 1px solid #413b3bff; border-radius: 3px; background-color: #e2edecff;">'
                                subjectOptions += '<option value="">-- Select Schema Subject to Describe --</option>'
                                getSchemaSubjects().each { subject ->
                                   subjectOptions += "<option value='${subject}'>${subject}</option>"
                                }
                                subjectOptions += '</select>'

                            return """
                              <div style="background-color: #f0f8ff; padding: 15px; border-radius: 5px; border-left: 4px solid #4169e1;">
                                  <h4 style="margin: 0 0 15px 0; color: #191970;">📋🔍 Describe Schema</h4>
                                  <table style="width: 100%; border-collapse: collapse;">
                                      <tr>
                                          <td style="padding: 8px; vertical-align: top; width: 200px;">
                                              <label style="font-weight: bold; color: #191970;">Subject Name *</label>
                                          </td>
                                          <td style="padding: 8px;">
                                               ${subjectOptions}
                                               <div style="font-size: 12px; color: #191970; margin-top: 3px;">Schema subject name to describe</div>
                                           </td>
                                       </tr>
                                       <tr>
                                           <td style="padding: 8px; vertical-align: top;">
                                              <label style="font-weight: bold; color: #191970;">Include Details</label>
                                           </td>
                                           <td style="padding: 8px;">
                                              <div style="margin-top: 5px;">
                                                   <label style="color: #191970; display: block; margin-bottom: 5px;">
                                                       <input type="checkbox" name="value" value="show_schema_version" checked style="margin-right: 5px;">
                                                       Show All version definition/content
                                                  </label>
                                              </div>
                                           </td>
                                        </tr>
                                    </table>
                                </div>
                            """
                        } else if (OPERATION == 'LIST_SCHEMA') {
                            return """
                                <div style="background-color: #f0fff0; padding: 15px; border-radius: 5px; border-left: 4px solid #32cd32;">
                                    <h4 style="margin: 0; color: #006400;">📋📝 List Schemas</h4>
                                    <p style="margin: 5px 0 15px 0; color: #006400;">This operation will list all registered schemas in the Schema Registry with their subjects, versions, and compatibility settings.</p>
                                    <table style="width: 100%; border-collapse: collapse;">
                                        <tr>
                                            <td style="padding: 8px; vertical-align: top;">
                                                <label style="font-weight: bold; color: #006400;">Include Details</label>
                                            </td>
                                            <td style="padding: 8px;">
                                                <div style="margin-top: 5px;">
                                                    <label style="color: #006400; display: block; margin-bottom: 5px;">
                                                        <input type="checkbox" name="value" value="show_versions" checked style="margin-right: 5px;">
                                                        Show all versions for each subject
                                                    </label>
                                                </div>
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
        SCHEMA_REGISTRY_URL = 'http://localhost:8081'
        SECURITY_PROTOCOL = 'SASL_PLAINTEXT'
        TOPICS_LIST_FILE = 'kafka-topics-list.txt'
        TOPIC_DESCRIPTION_FILE = 'kafka-topics-describe.txt'
        SCHEMA_LIST_FILE = 'schema-subjects-list.txt'
        SCHEMA_DESCRIPTION_FILE = 'schema-subject-description.txt'
        CONSUME_MESSAGES_FILE = 'consumed-messages.txt'
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
                            env.INCLUDE_INTERNAL = values.contains('true') ? 'true' : 'false'
                            echo "Listing all topics (Include internal: ${env.INCLUDE_INTERNAL})"
                            break
                        case 'ALTER_TOPIC':
                            env.TOPIC_NAME = values[0]
                            env.RETENTION_DAYS = values[1] ?: '7'
                            env.CLEANUP_POLICY = values[2].replace('|', ',') ?: 'delete'
                            env.SEGMENT_BYTES = values[3] ?: '1073741824'
                            env.MIN_INSYNC_REPLICAS = values[4] ?: '1'
                            env.MAX_MESSAGE_BYTES = values[5] ?: '1000000'
                            echo """
                            Altering topic: ${env.TOPIC_NAME}
                                  - Retention Days: ${env.RETENTION_DAYS}
                                  - Cleanup Policy: ${env.CLEANUP_POLICY}
                                  - Segment Bytes: ${env.SEGMENT_BYTES}
                                  - Min In-Sync Replicas: ${env.MIN_INSYNC_REPLICAS}
                                  - Max Message Bytes: ${env.MAX_MESSAGE_BYTES}
                            """
                            break
                        case 'CONSUMER':
                            env.TOPIC_NAME = values[0]
                            env.CONSUMER_GROUP_ID = values[1] ?: ''
                            env.OFFSET_RESET = values[2] ?: 'latest'
                            env.MAX_MESSAGES = values[3] ?: '10'
                            env.TIMEOUT_SECONDS = values[4] ?: '30'
                            env.MESSAGE_FORMAT = values[5] ?: 'JSON'
                            echo """Consumer configuration:
                                Topic: ${env.TOPIC_NAME}"
                                Consumer Group: ${env.CONSUMER_GROUP_ID}"
                                Offset Reset: ${env.OFFSET_RESET}"
                                Max Messages: ${env.MAX_MESSAGES}"
                                Timeout: ${env.TIMEOUT_SECONDS} seconds"
                                Message Format: ${env.MESSAGE_FORMAT}"
                            """
                            break
                        case 'PRODUCER':
                            values = option.split(',', 5).collect { it.trim() }
                            env.PRODUCER_TYPE = values[0] ?: 'standard'
                            env.TOPIC_NAME = values[1]
                            env.SCHEMA_SUBJECT = values[2] ?: ''
                            env.MESSAGE_COUNT = values[3] ?: '1'
                            env.MESSAGE_DATA = values[4] ?: ''

                            echo """Producer configuration:
                                Producer Type: ${env.PRODUCER_TYPE}
                                Topic: ${env.TOPIC_NAME}
                                Schema Subject: ${env.SCHEMA_SUBJECT}
                                Message Count: ${env.MESSAGE_COUNT}
                                Message DATA: ${env.MESSAGE_DATA}
                            """
                            break
                        case 'LIST_SCHEMA':
                            env.SHOW_VERSIONS = values.contains('true') ? 'true' : 'false'
                            echo "Listing all schemas (Show versions: ${env.SHOW_VERSIONS})"
                            break
                        case 'DELETE_SCHEMA':
                            env.SUBJECT_NAME = values[0]
                            echo "Delete Subject ${env.SUBJECT_NAME}"

                            break
                        case 'DESCRIBE_SCHEMA':
                            env.SUBJECT_NAME = values[0]
                            env.SHOW_VERSIONS = values[1].contains('true') ? 'true' : 'false'
                            echo "Describe schema ${env.SUBJECT_NAME} (Show versions: ${env.SHOW_VERSIONS})"
                            break
                        case 'REGISTER_SCHEMA':
                            echo "${params.TOPIC_OPTIONS}"
                            values = option.split(',', 5).collect { it.trim() }
                            env.TOPIC_NAME = values[0]
                            env.SCHEMA_FOR = values[1]
                            env.SCHEMA_TYPE = values[2]
                            env.SCHEMA_CONTENT = values[4].trim().replaceAll(/,\s*$/, '')
                            echo """
                            Register schema ${SCHEMA_FOR} in ${SCHEMA_TYPE} to topic : ${env.TOPIC_NAME}
                            With content ${SCHEMA_CONTENT}
                            """
                            break
                    }
                }
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'REGISTER_SCHEMA':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            if (!env.TOPIC_NAME.matches('^[a-zA-Z0-9._-]+$')) error "Invalid topic name format"
                            echo "✅ Validation passed"
                            break
                        case 'CREATE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            if (!env.TOPIC_NAME.matches('^[a-zA-Z0-9._-]+$')) error "Invalid topic name format"
                            echo "✅ Validation passed"
                            break
                        case 'CONSUMER':
                        case 'DESCRIBE_TOPIC':
                        case 'DELETE_TOPIC':
                            if (!env.TOPIC_NAME?.trim()) error "Topic name required"
                            echo "✅ Validation passed"
                            break
                        case 'DESCRIBE_SCHEMA':
                        case 'DELETE_SCHEMA':
                            if (!env.SUBJECT_NAME?.trim()) error "Topic name required"
                            echo "✅ Validation passed"
                            break
                        case 'LIST_TOPICS':
                        case 'LIST_SCHEMA':
                            echo "✅ Validation passed"
                            break
                    }
                }
            }
        }

        stage('Execute Operation') {
            steps {
                script {
                    switch(params.OPERATION) {
                        case 'LIST_TOPICS':
                            echo "==== Calling List Topic job ===="

                            def listTopicsJob = build job: 'org-cp-tools/CP-jenkins/list-topics',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    booleanParam(name: 'INCLUDE_INTERNAL', value: "${env.INCLUDE_INTERNAL}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                ],
                                propagate: false,
                                wait: true

                            copyArtifacts projectName: 'org-cp-tools/CP-jenkins/list-topics',
                                    buildNumber: "${listTopicsJob.number}",
                                    filter: 'kafka-topics-list.txt',
                                    target: '.'
                            break

                        case 'CREATE_TOPIC':
                            echo "==== Calling Create Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/create-topic',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'PARTITIONS', value: "${env.PARTITIONS}"),
                                    string(name: 'REPLICATION_FACTOR', value: "${env.REPLICATION_FACTOR}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}")
                                ],
                                propagate: false,
                                wait: true
                            echo "==== Update topic listed ===="
                             build job: 'org-cp-tools/CP-jenkins/list-topics',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DESCRIBE_TOPIC':
                            echo "==== Calling Describe Topic job ===="
                            def DescribeTopicsJob =  build job: 'org-cp-tools/CP-jenkins/describe-topic',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}")
                                ],
                                propagate: false,
                                wait: true

                            copyArtifacts projectName: 'org-cp-tools/CP-jenkins/describe-topic',
                                    buildNumber: "${DescribeTopicsJob.number}",
                                    filter: 'kafka-topics-describe.txt',
                                    target: '.'
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
                            build job: 'org-cp-tools/CP-jenkins/delete-topic',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER ?: 'localhost:9092'}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL ?: 'SASL_PLAINTEXT'}"),
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    booleanParam(name: 'CONFIRM_DELETE', value: true)
                                ],
                                propagate: true,
                                wait: true
                            echo "==== Update topic listed ===="
                             build job: 'org-cp-tools/CP-jenkins/list-topics',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                ],
                                propagate: false,
                                wait: true
                            break
                        case 'ALTER_TOPIC':
                            echo "==== Calling Alter Topic job ===="
                            build job: 'org-cp-tools/CP-jenkins/alter-topic',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'RETENTION_DAYS', value: "${env.RETENTION_DAYS}"),
                                    string(name: 'CLEANUP_POLICY', value: "${env.CLEANUP_POLICY}"),
                                    string(name: 'SEGMENT_BYTES', value: "${env.SEGMENT_BYTES}"),
                                    string(name: 'MIN_INSYNC_REPLICAS', value: "${env.MIN_INSYNC_REPLICAS}"),
                                    string(name: 'MAX_MESSAGE_BYTES', value: "${env.MAX_MESSAGE_BYTES}")

                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'PRODUCER':
                            echo "==== Calling Producer job ===="
                            build job: 'org-cp-tools/CP-jenkins/producer',
                                parameters: [
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                    string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                    text(name: 'MESSAGE_COUNT ', value: "${env.MESSAGE_COUNT }"),
                                    text(name: 'MESSAGE_DATA ', value: "${env.MESSAGE_DATA }"),

                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'CONSUMER':
                            if ("${env.MESSAGE_FORMAT}" == "JSON" ){
                                echo "==== Calling Consumer job ===="
                                def ConsumerJob = build job: 'org-cp-tools/CP-jenkins/consumer',
                                    parameters: [
                                        string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                        string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                        string(name: 'KAFKA_BOOTSTRAP_SERVER', value: "${env.KAFKA_BOOTSTRAP_SERVER}"),
                                        string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                        string(name: 'OFFSET_RESET', value: "${env.OFFSET_RESET}"),
                                        string(name: 'CONSUMER_GROUP_ID', value: "${env.CONSUMER_GROUP_ID}"),
                                        string(name: 'MAX_MESSAGES', value: "${env.MAX_MESSAGES}"),
                                        string(name: 'TIMEOUT_SECONDS', value: "${env.TIMEOUT_SECONDS}")
                                    ],
                                    propagate: false,
                                    wait: true
                                copyArtifacts projectName: 'org-cp-tools/CP-jenkins/consumer',
                                    buildNumber: "${ConsumerJob.number}",
                                    filter: 'consumed-messages.txt',
                                    target: '.'
                                break
                            } else {
                                echo "==== Calling Consumer job ===="
                                def ConsumerScheJob = build job: 'org-cp-tools/CP-jenkins/consumer-schema',
                                    parameters: [
                                        string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                        string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                        string(name: 'SECURITY_PROTOCOL', value: "${env.SECURITY_PROTOCOL}"),
                                        string(name: 'OFFSET_RESET', value: "${env.OFFSET_RESET}"),
                                        string(name: 'CONSUMER_GROUP_ID', value: "${env.CONSUMER_GROUP_ID}"),
                                        string(name: 'MAX_MESSAGES', value: "${env.MAX_MESSAGES}"),
                                        string(name: 'MESSAGE_FORMAT', value: "${env.MESSAGE_FORMAT}"),
                                        string(name: 'TIMEOUT_SECONDS', value: "${env.TIMEOUT_SECONDS}"),
                                        string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}")
                                    ],
                                    propagate: false,
                                    wait: true
                                copyArtifacts projectName: 'org-cp-tools/CP-jenkins/consumer-schema',
                                    buildNumber: "${ConsumerScheJob.number}",
                                    filter: 'consumed-messages.txt',
                                    target: '.'
                                break
                            }

                        case 'REGISTER_SCHEMA':
                            echo "==== Calling Register Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/register-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                    string(name: 'TOPIC_NAME', value: "${env.TOPIC_NAME}"),
                                    string(name: 'SCHEMA_FOR', value: "${env.SCHEMA_FOR}"),
                                    string(name: 'SCHEMA_TYPE', value: "${env.SCHEMA_TYPE}"),
                                    text(name: 'SCHEMA_CONTENT', value: "${env.SCHEMA_CONTENT}"),
                                ],
                                propagate: false,
                                wait: true
                            build job: 'org-cp-tools/CP-jenkins/list-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'DELETE_SCHEMA':
                            echo "==== Calling Delete Schema job ===="
                            build job: 'org-cp-tools/CP-jenkins/delete-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                    string(name: 'SUBJECT_NAME', value: "${env.SUBJECT_NAME}"),
                                ],
                                propagate: false,
                                wait: true
                            build job: 'org-cp-tools/CP-jenkins/list-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                ],
                                propagate: false,
                                wait: true
                            break

                        case 'LIST_SCHEMA':
                            echo "==== Calling List Schema job ===="
                            def listSchemasJob = build job: 'org-cp-tools/CP-jenkins/list-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                    booleanParam(name: 'INCLUDE_VERSIONS', value: "${env.SHOW_VERSIONS}")
                                ],
                                propagate: false,
                                wait: true

                            copyArtifacts projectName: 'org-cp-tools/CP-jenkins/list-schema',
                                    buildNumber: "${listSchemasJob.number}",
                                    filter: 'schema-subjects-list.txt',
                                    target: '.'
                            break

                        case 'DESCRIBE_SCHEMA':
                            echo "==== Calling List Schema job ===="
                            def describeSchemasJob = build job: 'org-cp-tools/CP-jenkins/list-schema',
                                parameters: [
                                    string(name: 'COMPOSE_DIR', value: "${env.COMPOSE_DIR}"),
                                    string(name: 'SCHEMA_REGISTRY_URL', value: "${env.SCHEMA_REGISTRY_URL}"),
                                    string(name: 'SUBJECT_NAME', value: "${env.SUBJECT_NAME}") ,
                                    booleanParam(name: 'INCLUDE_VERSIONS', value: "${env.SHOW_VERSIONS}")
                                ],
                                propagate: false,
                                wait: true

                            copyArtifacts projectName: 'org-cp-tools/CP-jenkins/list-schema',
                                    buildNumber: "${describeSchemasJob.number}",
                                    filter: 'schema-subject-description.txt',
                                    target: '.'
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

                def operationFiles = [
                    'LIST_TOPICS': env.TOPICS_LIST_FILE,
                    'DESCRIBE_TOPIC': env.TOPIC_DESCRIPTION_FILE,
                    'LIST_SCHEMA': env.SCHEMA_LIST_FILE,
                    'DESCRIBE_SCHEMA': env.SCHEMA_DESCRIPTION_FILE,
                    'CONSUMER': env.CONSUME_MESSAGES_FILE,
                ]

                def currentFile = operationFiles[params.OPERATION]

                if (currentFile) {
                    if (fileExists(currentFile)) {
                        echo "📄 ${params.OPERATION} Results:"
                        echo "=" * 60
                        echo readFile(currentFile)
                        echo "=" * 60

                        archiveArtifacts artifacts: currentFile,
                                       fingerprint: true,
                                       allowEmptyArchive: true
                        echo "📦 Results archived as artifact: ${currentFile}"
                    } else {
                        echo "⚠️ Warning: Output file ${currentFile} not found"
                    }
                }

                echo "✅ Kafka topic operation '${params.OPERATION}' completed successfully"
            }
        }
        failure {
            echo "❌ Kafka topic operation '${params.OPERATION}' failed - check logs for details"
        }
    }
}


/*
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
*/