# Wazuh-Artificial-Intelligence
Hello, this repository has details about use AI Claude Haiku on Wazuh Dashboard.

I hope that it help you!

This test was make on Wazuh 4.9.1, this version use as base Opensearch 2.13

To make this it's necessary install some plugins on Wazuh.

You're gonna do download this of plugins on this:

https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.13.0/opensearch-dashboards-2.13.0-linux-x64.tar.gz

Linux Command:

curl https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.13.0/opensearch-dashboards-2.13.0-linux-x64.tar.gz -o opensearch-dashboards.tar.gz

After that, you'll descompress the file:

tar -xvzf opensearch-dashboards.tar.gz

So, you'll copy plugins to directories:

cp -r opensearch-dashboards-2.13.0/plugins/opensearch-observability/ /usr/share/wazuh-dashboard/plugins/

cp -r opensearch-dashboards-2.13.0/plugins/opensearch-ml/ /usr/share/wazuh-dashboard/plugins/

cp -r opensearch-dashboards-2.13.0/plugins/assistantDashboards/ /usr/share/wazuh-dashboard/plugins/

You need to give permission to plugins:

chown -R wazuh-dashboard:wazuh-dashboard /usr/share/wazuh-dashboard/plugins/<PLUGIN_NAME>/

chmod -R 750 /usr/share/wazuh-dashboard/plugins/<PLUGIN_NAME>/ 

On /etc/wazuh-dashboard/opensearch_dashboards.yml you'll edit and add the lines:

assistant.chat.enabled: true

observability.query_assist.enabled: true

On Wazuh Indexer you need to install two plugins, you can install using Maven coordinates

Access directory:

cd /usr/share/wazuh-indexer/

./bin/opensearch-plugin install org.opensearch.plugin:opensearch-flow-framework:2.13.0.0

./bin/opensearch-plugin install org.opensearch.plugin:opensearch-skills:2.13.0.0

I had a problem with assistantDashboards plugin, so to resolve it was necessary:

Install abortcontroller with this command:

npm install abort-controller

Edit the file and add this code:

nano /usr/share/wazuh-dashboard/plugins/assistantDashboards/server/services/chat/olly_chat_service.js

"use strict";

if (typeof AbortController === 'undefined') {
  const { AbortController } = require('abort-controller');
  global.AbortController = AbortController;
}

Ok! Now, you go to console.

On DevTools you'll execute the commands:

PUT /_cluster/settings
{
  "persistent" : {
    "plugins.ml_commons.only_run_on_ml_node":"false"
  }
}

You'll create a connector to an externally hosted model:

POST /_plugins/_ml/connectors/_create
{
    "name": "Amazon Bedrock Claude Haiku",
    "description": "Connector for Amazon Bedrock Claude Haiku",
    "version": 1,
    "protocol": "aws_sigv4",
    "credential": {
      "access_key": "****************",
      "secret_key": "**************************************"
    },
    "parameters": {
        "region": "us-east-1",
        "service_name": "bedrock",
        "auth": "Sig_V4",
        "response_filter": "$.content[0].text",
        "max_tokens_to_sample": "8000",
        "anthropic_version": "bedrock-2023-05-31",
        "model": "anthropic.claude-3-haiku-20240307-v1:0"
    },
    "actions": [
        {
            "action_type": "predict",
            "method": "POST",
            "headers": {
                "content-type": "application/json"
            },
            "url": "https://bedrock-runtime.us-east-1.amazonaws.com/model/${parameters.model}/invoke"
,
            "request_body": "{\"messages\":[{\"role\":\"user\",\"content\":[{\"type\":\"text\",\"text\":\"${parameters.prompt}\"}]}],\"anthropic_version\":\"${parameters.anthropic_version}\",\"max_tokens\":${parameters.max_tokens_to_sample}}"
        }
    ]
}

![image](https://github.com/user-attachments/assets/42075e75-1580-4f57-9ae7-b58fbd6fc21d)


Register and deploy the externally hosted model

POST /_plugins/_ml/model_groups/_register
{
    "name": "AWS Bedrock",
    "description": "This is a public model group"
}

![image](https://github.com/user-attachments/assets/4f593bb2-438a-47ab-a0b2-a1108bf6aa5d)


Next, register and deploy the externally hosted Claude mode:

POST /_plugins/_ml/models/_register?deploy=true
{
    "name": "Bedrock Claude V2 model",
    "function_name": "remote",
    "model_group_id": "43J12JIB2Zh-SJAsdAkj",
    "description": "Test Model",
    "connector_id": "5HJ22JIB2Zh-SJAsyAlT"
}

![image](https://github.com/user-attachments/assets/58eca8fa-8f6b-4e91-9294-43521f25e2d3)


To test the LLM, send the following predict request:

POST /_plugins/_ml/models/5nJ32JIB2Zh-SJAsXglb/_predict
{
  "parameters": {
    "prompt": "\n\nHuman:hello\n\nAssistant:"
  }
}

![image](https://github.com/user-attachments/assets/3d1e318d-9fd5-4bfc-a0e1-5ccf26b105d6)


POST /_plugins/_ml/agents/_register
{
  "name": "Test_Agent_For_ReAct_ClaudeV2",
  "type": "conversational",
  "description": "this is a test agent",
  "llm": {
    "model_id": "6OgZoZIB6Y9VmGpsp98r",
    "parameters": {
      "max_iteration": 5,
      "stop_when_no_tool_found": true
    }
  },
  "memory": {
    "type": "conversation_index"
  },
  "tools": [
    {
      "type": "MLModelTool",
      "name": "bedrock_claude_model",
      "description": "A general tool to answer any question",
      "parameters": {
        "model_id": "6OgZoZIB6Y9VmGpsp98r",
        "prompt": "Human: You're a cybersecurity analyst and you're gonna help me with cybersecurity logs.\n\n${parameters.chat_history:-}\n\nHuman: ${parameters.question}\n\nAssistant:"
      }
    }
  ]
}

![image](https://github.com/user-attachments/assets/8621cb55-98fe-4e96-b561-d0e97cb599df)


Test agent:

POST _plugins/_ml/agents/6nJ62JIB2Zh-SJAsSQn1/_execute
{
  "parameters": {
    "question": "Who are you?",
    "verbose": false
  }
}

![image](https://github.com/user-attachments/assets/86debab2-1b35-4db7-ab61-e8685d2fd7fa)


Now, you'll add the agent on interface:

PUT .plugins-ml-config/_doc/os_chat
{
    "type":"os_chat_root_agent",
    "configuration":{
        "agent_id": "6nJ62JIB2Zh-SJAsSQn1"
    }
}

![image](https://github.com/user-attachments/assets/36c9097b-a7c2-431d-8e31-14e2e7788dc6)

![image](https://github.com/user-attachments/assets/790cc142-80de-4f73-82f4-3c805ef77cff)

Congratulations, now you have a AI on Wazuh Console!!

Reference: https://opensearch.org/docs/latest/ml-commons-plugin/



