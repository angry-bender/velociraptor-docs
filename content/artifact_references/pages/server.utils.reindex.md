---
title: Server.Utils.ReIndex
hidden: true
tags: [Client Artifact]
---

This utility artifact replays all collected Generic.Client.Info
collections to the interrogation service forcing a reindex.

It should normally not be needed.


```yaml
name: Server.Utils.ReIndex
description: |
  This utility artifact replays all collected Generic.Client.Info
  collections to the interrogation service forcing a reindex.

  It should normally not be needed.

sources:
  - query: |
      SELECT * FROM foreach(row={
        SELECT Name AS ClientId
        FROM glob(globs="/clients/*", accessor="fs")
        WHERE Name =~ "^C."
      }, query={
        SELECT session_id,
             send_event(artifact="System.Flow.Completion",
                        row=dict(
                           Flow=dict(artifacts_with_results=artifacts_with_results),
                           ClientId=ClientId,
                           FlowId=session_id)) AS Event
        FROM flows(client_id=ClientId)
        WHERE request.artifacts =~ "Generic.Client.Info"
        LIMIT 1
      }, workers=10)

```
