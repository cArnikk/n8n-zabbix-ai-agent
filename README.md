
# N8N Agent AI with Zabbix



asd
# Prerequisites
In my scenario I will use two LXC containers, one for the Zabbix Server 7.0 and the other for the N8N instance, you can use the docker solution if you prefer

## LXC - N8N
By default N8N creates a webhook with a localhost address, we need to change this if we want to send any webhook to N8N. 


Edit N8N service configuration file

```bash
  /etc/systemd/system/n8n.service
```

and in the [service] section add environment with webhook
```bash
  Environment="WEBHOOK_URL=http://<n8n-address>:5678"
```
Then reload systemctl daemon and restart the n8n service
```bash
  systemctl daemon-reload
  systemctl restart n8n.service
```
## Zabbix Server

For now, there is no native solution to send alarm via webhook from Zabbix to N8N, so we need to do it by yourself.

First, we need to create new Media type in Zabbix, to do this, go to Alerts -> Media types

![image](https://github.com/user-attachments/assets/bd189091-87f2-4cd6-b2eb-77cc05793393)

then, Create media type 
![image](https://github.com/user-attachments/assets/e4f164f8-b03d-44bc-aa84-da26b8c9997a)

Provide all necessary information for new webhook:

|Name|N8N|
|--- | ---|

|Type |Webhook|
|--- | ---|

| Name     | Value      |
| ---      | ---       |
| EVENTID        |{EVENT.ID}|
| EVENT_DATE     |{EVENT.DATE}|
| EVENT_TIME     |{EVENT.TIME}|
| HOSTNAME     |{HOST.NAME}|
| ITEM_KEY     |{ITEM.KEY}|
| ITEM_VALUE     |{ITEM.VALUE}|
| PROBLEM_URL     |http://\<zabbix-address>/tr_events.php?triggerid={TRIGGER.ID}&eventid={EVENT.ID}|
| TRIGGER_NAME     |{TRIGGER.NAME}|
| TRIGGER_SEVERITY     |{TRIGGER.SEVERITY}|
| TRIGGER_STATUS     |{TRIGGER.STATUS}|


Script:

```javascript
  try {
    Zabbix.Log(4, '[n8n Webhook] Received parameters: ' + value);
    var n8nWebhookUrl = '<N8N-webhook-address>';
    var params = JSON.parse(value);

    var payload = {
        zabbix_event_id: params.EVENTID || 'N/A',
        host: params.HOSTNAME || 'N/A',
        trigger: params.TRIGGER_NAME || params.SUBJECT || 'N/A',
        severity: params.TRIGGER_SEVERITY || 'N/A',
        status: params.TRIGGER_STATUS || 'N/A',
        event_time: params.EVENT_TIME || 'N/A',
        event_date: params.EVENT_DATE || 'N/A',
		item_key: params.ITEM_KEY || 'N/A',
        item_value: params.ITEM_VALUE || 'N/A',
        problem_url: params.PROBLEM_URL || ''
    };

    Zabbix.Log(4, '[n8n Webhook] Payload to send: ' + JSON.stringify(payload));

    var request = new HttpRequest();
    request.addHeader('Content-Type: application/json');
    var response = request.post(n8nWebhookUrl, JSON.stringify(payload));

    Zabbix.Log(4, '[n8n Webhook] Response received: ' + response);

    if (request.getStatus() >= 200 && request.getStatus() < 300) {
        Zabbix.Log(4, '[n8n Webhook] Successfully sent data to n8n. Status: ' + request.getStatus());
        return JSON.stringify({
            status: 'success',
            http_status: request.getStatus(),
            response: response
        });
    } else {
        Zabbix.Log(1, '[n8n Webhook] Failed to send data to n8n. Status: ' + request.getStatus() + ' Response: ' + response);
        throw 'Failed to send data to n8n. HTTP Status: ' + request.getStatus() + ' Response: ' + response;
    }

} catch (error) {
    Zabbix.Log(1, '[n8n Webhook] Execution error: ' + error);
    throw 'n8n webhook script execution failed: ' + error;
}
```

![image](https://github.com/user-attachments/assets/f6d23a95-5294-49fa-9573-90a52a921e5c)

For message template you can use default message template from Zabbix.

![image](https://github.com/user-attachments/assets/97ba03bd-5358-4c1d-963b-e000ea647f59)

Now, create Trigger action to execute webhook for N8N

