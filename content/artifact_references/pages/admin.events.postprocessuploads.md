---
title: Admin.Events.PostProcessUploads
hidden: true
tags: [Server Event Artifact]
---

Sometimes we would like to post process uploads collected as part of
the hunt's artifact collections

Post processing means to watch the hunt for completed flows and run
a post processing command on the files obtained from each host.

The command will receive the list of paths of the files uploaded by
the artifact. We dont actually care what the command does with those
files - we will just relay our stdout/stderr to the artifact's
result set.


```yaml
name: Admin.Events.PostProcessUploads
description: |
  Sometimes we would like to post process uploads collected as part of
  the hunt's artifact collections

  Post processing means to watch the hunt for completed flows and run
  a post processing command on the files obtained from each host.

  The command will receive the list of paths of the files uploaded by
  the artifact. We dont actually care what the command does with those
  files - we will just relay our stdout/stderr to the artifact's
  result set.

type: SERVER_EVENT

required_permissions:
  - EXECVE

parameters:
  - name: uploadPostProcessCommand
    description: |
      The command to run - must be a json array of strings! The list
      of files will be appended to the end of the command.
    default: |
      ["/bin/ls", "-l"]

  - name: uploadPostProcessArtifact
    description: |
      The name of the artifact to watch.
    default: Windows.Registry.NTUser.Upload

sources:
  - query: |
        LET files = SELECT Flow,
            array(a1=parse_json_array(data=uploadPostProcessCommand),
                  a2=file_store(path=Flow.uploaded_files)) as Argv
        FROM watch_monitoring(artifact='System.Flow.Completion')
        WHERE uploadPostProcessArtifact in Flow.artifacts_with_results

        SELECT * from foreach(
          row=files,
          query={
             SELECT Flow.session_id as FlowId, Argv,
                    Stdout, Stderr, ReturnCode
             FROM execve(argv=Argv)
          })

```
