## N8N AI Agent for Zabbix Alert Enrichment and Initial Triage

This project outlines a solution to empower Level 1 (L1) and Level 2 (L2) support teams to perform initial diagnostics and basic verification of Zabbix alerts using an AI-powered N8N workflow. The primary goal is to reduce the immediate involvement of Level 3 (L3) teams by providing enriched alert information and suggested troubleshooting steps.

**Important Security Consideration:** Before implementing this project, verify your company's policy regarding the transmission of potentially sensitive data (e.g., hostnames, IP addresses, tags, proxy names) to external AI services. If necessary, implement data anonymization techniques or explore the possibility of using an on-premise or private AI solution. Consult your security team for guidance.

## Guiding Principles for AI Agent Interaction

To ensure effective and consistent responses from the AI agent, consider the following when designing your prompts:

* **Define the AI's Role:** Clearly state what the AI's persona is (e.g., "You are an expert IT operations assistant").
* **Specify its Function:** Describe what the AI does within the IT team (e.g., "Your role is to analyze Zabbix alerts, provide context, and suggest initial troubleshooting steps").
* **Establish Behavior Rules:** Set guidelines for the AI's responses (e.g., "Provide concise, actionable information. Always maintain a professional and helpful tone.").
* **Provide Example Output:** Show the AI the desired format and content for its responses, including suggested solutions.
* **Maximize Information Input:** Supply the AI with as much relevant data from the Zabbix alert as possible.
* **Clearly State the Problem:** Ensure the prompt clearly communicates the nature of the alert and the desired output from the AI.

# Prerequisites

This guide assumes a setup using two LXC containers: one for the Zabbix Server (version 7.0 in this example) and another for the N8N instance.

You can leverage existing community scripts for Proxmox to simplify the installation of Zabbix Server and N8N in LXC containers:
* [ProxmoxVE Community Scripts](https://github.com/community-scripts/ProxmoxVE)

## N8N Instance Configuration: Enabling External Webhook Access

By default, N8N generates webhook URLs that are only accessible from `localhost`. To allow Zabbix to send alerts to N8N, you need to configure N8N to use an externally accessible webhook URL.

1.  **Edit the N8N Service Configuration File:**
    Open the N8N service file using a text editor (e.g., `nano` or `vim`):
    ```bash
    sudo vim /etc/systemd/system/n8n.service
    ```

2.  **Set the `WEBHOOK_URL` Environment Variable:**
    In the `[Service]` section of the file, add the following line. Replace `<n8n-external-ip-or-hostname>` with the actual IP address or hostname of your N8N instance and `<port>` with the N8N port (typically `5678`):
    ```bash
    Environment="WEBHOOK_URL=http://<n8n-external-ip-or-hostname>:<port>"
    ```
    **Example:**
    ```bash
    Environment="WEBHOOK_URL=http://192.168.1.100:5678"
    ```

3.  **Reload Systemd and Restart N8N:**
    Apply the changes and restart the N8N service:
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart n8n.service
    ```

## Zabbix Server Configuration: Creating a Webhook Media Type for N8N

Zabbix does not have a native integration for sending alerts to N8N via webhooks. Therefore, we need to create a custom Media Type.

1.  **Navigate to Media Types:**
    In the Zabbix frontend, go to **Alerts** -> **Media types**.

    ![Zabbix Menu: Alerts -> Media types](https://github.com/user-attachments/assets/25be0cda-8016-4fb2-b3ef-11727e84336f)

2.  **Create a New Media Type:**
    Click the "**Create media type**" button.

    ![Zabbix: Create media type button](https://github.com/user-attachments/assets/e4f164f8-b03d-44bc-aa84-da26b8c9997a)

3.  **Configure the Webhook Media Type:**
    Fill in the following details for the new N8N webhook:

    ![Zabbix: Media type configuration for N8N](https://github.com/user-attachments/assets/ebc96f22-0fd8-45ee-b941-73208233cbf3)

    * **Name:** `N8N` (or a descriptive name of your choice)
    * **Type:** `Webhook`
    * **Parameters:** Define the following parameters that will be sent in the webhook payload. Zabbix macros will populate these values dynamically.

        | Parameter Name     | Value                                                                  | Description                                       |
        | :----------------- | :--------------------------------------------------------------------- | :------------------------------------------------ |
        | `EVENTID`          | `{EVENT.ID}`                                                           | The unique ID of the event.                       |
        | `EVENT_DATE`       | `{EVENT.DATE}`                                                         | The date the event occurred.                      |
        | `EVENT_TIME`       | `{EVENT.TIME}`                                                         | The time the event occurred.                      |
        | `HOSTNAME`         | `{HOST.NAME}`                                                          | The name of the host that triggered the event.    |
        | `ITEM_KEY`         | `{ITEM.KEY}`                                                           | The key of the item that triggered the event.     |
        | `ITEM_VALUE`       | `{ITEM.VALUE}`                                                         | The value of the item that triggered the event.   |
        | `PROBLEM_URL`      | `http://<zabbix-server-address>/tr_events.php?triggerid={TRIGGER.ID}&eventid={EVENT.ID}` | A direct URL to the event in Zabbix. **Replace `<zabbix-server-address>` with your Zabbix server's address.** |
        | `TRIGGER_NAME`     | `{TRIGGER.NAME}`                                                       | The name of the trigger.                          |
        | `TRIGGER_SEVERITY` | `{TRIGGER.SEVERITY}`                                                   | The severity of the trigger.                      |
        | `TRIGGER_STATUS`   | `{TRIGGER.STATUS}`                                                     | The status of the trigger (PROBLEM or OK).       |

    * **Script:**
        Paste the following JavaScript code into the script field. This script constructs the JSON payload and sends it to your N8N webhook URL.

        **Important:** Replace `<N8N-webhook-address>` in the script with the actual production webhook URL you obtain from your N8N workflow (see N8N Configuration section below).

        ```javascript
        try {
            Zabbix.Log(4, '[N8N Webhook] Received parameters: ' + value);

            // **IMPORTANT:** Replace with your N8N Production Webhook URL
            var n8nWebhookUrl = '<N8N-webhook-address>';
            var params = JSON.parse(value);

            var payload = {
                zabbix_event_id: params.EVENTID || 'N/A',
                host: params.HOSTNAME || 'N/A',
                trigger: params.TRIGGER_NAME || params.SUBJECT || 'N/A', // params.SUBJECT can be a fallback
                severity: params.TRIGGER_SEVERITY || 'N/A',
                status: params.TRIGGER_STATUS || 'N/A',
                event_time: params.EVENT_TIME || 'N/A',
                event_date: params.EVENT_DATE || 'N/A',
                item_key: params.ITEM_KEY || 'N/A',
                item_value: params.ITEM_VALUE || 'N/A',
                problem_url: params.PROBLEM_URL || ''
            };

            Zabbix.Log(4, '[N8N Webhook] Payload to send: ' + JSON.stringify(payload));

            var request = new HttpRequest();
            request.addHeader('Content-Type: application/json');

            var response = request.post(n8nWebhookUrl, JSON.stringify(payload));

            Zabbix.Log(4, '[N8N Webhook] Response received: ' + response);

            if (request.getStatus() >= 200 && request.getStatus() < 300) {
                Zabbix.Log(4, '[N8N Webhook] Successfully sent data to N8N. Status: ' + request.getStatus());
                return JSON.stringify({
                    status: 'success',
                    http_status: request.getStatus(),
                    response: response
                });
            } else {
                Zabbix.Log(1, '[N8N Webhook] Failed to send data to N8N. Status: ' + request.getStatus() + ' Response: ' + response);
                throw 'Failed to send data to N8N. HTTP Status: ' + request.getStatus() + ' Response: ' + response;
            }

        } catch (error) {
            Zabbix.Log(1, '[N8N Webhook] Execution error: ' + error);
            throw 'N8N webhook script execution failed: ' + error;
        }
        ```

    * **Message Templates:**
        You can typically use the default message template or customize it if needed. For this integration, the core data is sent via the script parameters.

        ![Zabbix: Media type message template](https://github.com/user-attachments/assets/97ba03bd-5358-4c1d-963b-e000ea647f59)

4.  **Save the Media Type.** Click "Add" or "Update" to save your new media type.

## Zabbix Server Configuration: Creating a Trigger Action

Next, create a trigger action to specify when and how Zabbix should use the N8N media type.

1.  **Navigate to Trigger Actions:**
    Go to **Alerts** -> **Actions** -> **Trigger actions**.

    ![Zabbix Menu: Alerts -> Actions -> Trigger actions](https://github.com/user-attachments/assets/69489fde-2bed-487e-b131-984526e58b4a)

2.  **Create a New Action:**
    Click the "**Create action**" button in the top right corner.

3.  **Configure the Action - "Action" Tab:**
    * **Name:** Provide a descriptive name (e.g., "Send Alert to N8N AI Agent").
    * **Conditions:** Define the conditions under which this action will be triggered (e.g., specific trigger severities, host groups, problem status, etc.). A common condition is `Trigger severity >= "Warning"`.
    * **Enabled:** Ensure the checkbox is checked.

    ![Zabbix: Trigger action configuration - Action tab](https://github.com/user-attachments/assets/c239a809-522d-4d3a-975d-5ca9441a8273)

4.  **Configure the Action - "Operations" Tab:**
    * In the "Operations" block, click "**Add**".
    * **Operation Details:**
        * **Send to Users:** Select a Zabbix user that has the N8N Media Type configured in their user profile (under "Media").
        * **Send to media type:** From the dropdown, choose the "N8N" media type you created.
    * Click "**Add**" to save the operation.

    ![Zabbix: Trigger action configuration - Operations tab](https://github.com/user-attachments/assets/549388b1-d0c8-487f-ad1b-7d40017601d6)

5.  **Save the Trigger Action.** Click "Add" or "Update" at the bottom of the page.

## N8N Workflow Configuration

This section describes an example N8N workflow that receives Zabbix alerts, processes them with an AI agent (Gemini in this example), stores information (optional), and sends an email notification. You can adapt this workflow to your specific needs and preferred tools.

**Example Workflow Overview:**

![N8N Workflow: Zabbix to AI to Email](https://github.com/user-attachments/assets/81b931a4-4660-42ff-956e-055eea551917)

1.  **Webhook Node (Trigger):**
    * This node is the entry point for Zabbix alerts.
    * **Webhook URLs:** N8N provides a **Test URL** and a **Production URL**.
        * Use the **Test URL** for initial setup and testing your Zabbix media type script.
        * **Crucially, once you've tested your Zabbix media type script and N8N workflow, update the `n8nWebhookUrl` variable in your Zabbix Media Type script to use the N8N Production URL.**
    * **HTTP Method:** Ensure this is set to `POST`.
    * **Authentication:** Set to `None` if your Zabbix script doesn't send authentication headers (as in the example).

    ![N8N Webhook Node: Test and Production URLs](https://github.com/user-attachments/assets/574b2799-09e3-46c9-9743-7b3807936676)

2.  **Edit Fields Node (Optional but Recommended):**
    * This node allows you to extract and rename fields from the incoming JSON payload from Zabbix for easier use in subsequent nodes.
    * Map the fields received from Zabbix (e.g., `body.zabbix_event_id`, `body.host`, `body.trigger`) to more user-friendly names if desired (e.g., `eventId`, `hostname`, `triggerName`). The exact path will depend on how the Webhook node structures the incoming data (usually under `body`).

    ![N8N Edit Fields Node: Extracting Zabbix data](https://github.com/user-attachments/assets/d315900f-44de-49e8-ae11-9795a47aa3f5)

3.  **AI Agent Node (e.g., Gemini, OpenAI, Anthropic Claude, etc.):**
    * This node sends the alert information to your chosen AI model.
    * **Prompt Design:**
        * **System Message (or equivalent field in the AI node):** Define the AI's role, behavior, and any overarching rules (as discussed in "Guiding Principles for AI Agent Interaction").
        * **User Message (Prompt):** Construct a prompt using the data extracted from the Zabbix alert (e.g., from the "Edit Fields" node or directly from the Webhook node's output). Ask the AI to analyze the alert and provide insights or troubleshooting steps.
        * **Example Prompt Structure (using N8N expressions):**
            ```text
            You are an L1 IT Support Agent. Analyze the following Zabbix alert and provide a brief summary of the problem, potential causes, and 2-3 basic troubleshooting steps.

            Alert Details:
            Host: {{ $json.host }}
            Trigger: {{ $json.trigger }}
            Severity: {{ $json.severity }}
            Item Value: {{ $json.item_value }}
            Item Key: {{ $json.item_key }}
            Event Time: {{ $json.event_time }}
            Zabbix Event Link: {{ $json.problem_url }}

            Suggest concrete actions the L1 team can take.
            Return your response in a clear, easy-to-read format.
            ```
    * **Test Step:** Use the "Test step" or "Execute Node" functionality in N8N to verify the AI's response based on sample data from the previous node.

    ![N8N AI Agent Node: Prompt configuration](https://github.com/user-attachments/assets/9645dbdc-1485-4903-8a55-1d397e3725bf)

4.  **Data Store Node (Optional, e.g., Simple Store, Google Sheets, Database Node):**
    * Use this node to log the alert, the AI's response, and any other relevant information for auditing, reporting, or future analysis.
    * In the example image, "Simple Store" is used. You could integrate with nodes for MongoDB, PostgreSQL, Redis, Google Sheets, etc.

5.  **Notification Node (e.g., Email, Slack, Microsoft Teams, Discord):**
    * This node sends the enriched alert information (including the AI's suggestions) to the appropriate team or communication channel.
    * **Configure SMTP/Integration:** Set up your email account (SMTP), or authenticate with the API for services like Slack or Teams.
    * **Format Message:** Create a clear and informative message body using data from previous nodes (Zabbix alert details and the AI's response). Use N8N expressions to insert dynamic content.

    ![N8N Email Node: Message configuration](https://github.com/user-attachments/assets/4a1de0fe-bfb4-4b8f-bf4a-aa17081c9adf)

**Example Output (Email/Notification):**

This is an example of what the final notification might look like, combining Zabbix alert data with the AI's analysis and suggestions.

![Example Email Notification with AI analysis](https://github.com/user-attachments/assets/2219b286-d2a4-4c33-b2b8-bb16ff4ab09b)

## Conclusion

By integrating Zabbix with an N8N AI-powered workflow, you can significantly enhance your IT operations team's ability to respond to alerts. This solution provides valuable context and initial troubleshooting steps, enabling L1/L2 teams to resolve more issues independently and reducing the burden on L3 support.
