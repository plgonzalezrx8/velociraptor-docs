---
title: Server.Alerts.Trackaccount
hidden: true
tags: [Server Event Artifact]
---

This artifact alerts when account usage of a monitored account is detected. This is a server-side artifact, please note that it requires the client_event artifact 'Windows.Events.Trackaccount' to be enabled.


```yaml
name: Server.Alerts.Trackaccount
description: |
   This artifact alerts when account usage of a monitored account is detected. This is a server-side artifact, please note that it requires the client_event artifact 'Windows.Events.Trackaccount' to be enabled.

author: Jos Clephas - @DfirJos

type: SERVER_EVENT

parameters:
  - name: SlackToken
    description: The token URL obtained from Slack/Teams/Discord (or basicly any communication-service that supports webhooks). Leave blank to use server metadata. e.g. https://hooks.slack.com/services/XXXX/YYYY/ZZZZ

sources:
  - query: |
        LET token_url = if(
           condition=SlackToken,
           then=SlackToken,
           else=server_metadata().SlackToken)

        LET hits = SELECT * from watch_monitoring(artifact='Windows.Events.Trackaccount')

        SELECT * FROM foreach(row=hits,
        query={
           SELECT EventRecordID, EventID, TargetUserName, TargetWorkstationName, SourceComputer, LogonType, EventTime, ClientId, Url, Content, Response FROM http_client(
            data=serialize(item=dict(
                text=format(format="EventID: %v - Account '%v' authenticated from system '%v' to '%v' with LogonType %v at %v on client %v (EventRecordID: %v)",
                            args=[EventID, TargetUserName, TargetWorkstationName, SourceComputer, LogonType, EventTime, ClientId, EventRecordID])),
                format="json"),
            headers=dict(`Content-Type`="application/json"),
            method="POST",
            url=token_url)
        })

```
