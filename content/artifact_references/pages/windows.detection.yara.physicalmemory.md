---
title: Windows.Detection.Yara.PhysicalMemory
hidden: true
tags: [Client Artifact]
---

This artifact enables running Yara over physical memory.

There are 2 kinds of Yara rules that can be deployed:
 &nbsp;1. Url link to a yara rule.
 &nbsp;2. A Standard Yara rule attached as a parameter.

Only one method of Yara will be applied and search order is as above. The
default is Cobalt Strike opcodes.

The artifact will load the winpmem driver, then yara scan the
physical memory and remove the driver.

NOTE: This artifact is experimental and can crash the system!


```yaml
name: Windows.Detection.Yara.PhysicalMemory
description: |
  This artifact enables running Yara over physical memory.

  There are 2 kinds of Yara rules that can be deployed:
   &nbsp;1. Url link to a yara rule.
   &nbsp;2. A Standard Yara rule attached as a parameter.

  Only one method of Yara will be applied and search order is as above. The
  default is Cobalt Strike opcodes.

  The artifact will load the winpmem driver, then yara scan the
  physical memory and remove the driver.

  NOTE: This artifact is experimental and can crash the system!

tools:
  - name: WinPmem
    url: https://github.com/Velocidex/WinPmem/releases/download/v4.0.rc1/winpmem_mini_x64_rc2.exe
    serve_locally: true

type: CLIENT
parameters:
  - name: Context
    description: How many bytes of context around the hit to return
    type: int
    default: "0"
  - name: YaraUrl
    description: If configured will attempt to download Yara rules from Url
    type: upload
  - name: YaraRule
    type: yara
    description: Final Yara option and the default if no other options provided.
    default: |
      rule win_cobalt_strike_auto {
         meta:
           author = "Felix Bilstein - yara-signator at cocacoding dot com"
           date = "2019-11-26"
           version = "1"
           description = "autogenerated rule brought to you by yara-signator"
           tool = "yara-signator 0.2a"
           malpedia_reference = "https://malpedia.caad.fkie.fraunhofer.de/details/win.cobalt_strike"
           malpedia_license = "CC BY-SA 4.0"
           malpedia_sharing = "TLP:WHITE"

         strings:
           $sequence_0 = { 3bc7 750d ff15???????? 3d33270000 }
           $sequence_1 = { e9???????? eb0a b801000000 e9???????? }
           $sequence_2 = { 8bd0 e8???????? 85c0 7e0e }
           $sequence_3 = { ffb5f8f9ffff ff15???????? 8b4dfc 33cd e8???????? c9 c3 }
           $sequence_4 = { e8???????? e9???????? 833d?????????? 7505 e8???????? }
           $sequence_5 = { 250000ff00 33d0 8b4db0 c1e908 }
           $sequence_6 = { ff75f4 ff7610 ff761c ff75fc }
           $sequence_7 = { 8903 6a06 eb39 33ff 85c0 762b 03f1 }
           $sequence_8 = { 894dd4 8b458c d1f8 894580 8b45f8 c1e818 0fb6c8 }
           $sequence_9 = { 890a 8b4508 0fb64804 81e1ff000000 c1e118 8b5508 0fb64205 }
           $sequence_10 = { 33d2 e8???????? 48b873797374656d3332 4c8bc7 488903 49ffc0 }
           $sequence_11 = { 488bd1 498d4bd8 498943e0 498943e8 }
           $sequence_12 = { b904000000 486bc90e 488b542430 4c8b442430 418b0c08 8b0402 }
           $sequence_13 = { ba80000000 e8???????? 488d4c2438 e8???????? 488d4c2420 8bd0 e8???????? }
           $sequence_14 = { 488b4c2430 8b0401 89442428 b804000000 486bc004 }
           $sequence_15 = { 4883c708 4883c304 49ffc3 48ffcd 0f854fffffff 488d4c2420 }

        condition:
            7 of them
      }

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      -- check which Yara to use
      LET yara_rules <= YaraUrl || YaraRule

      LET SparsePath = pathspec(
           DelegateAccessor='raw_file',
           DelegatePath='''\\.\pmem''',
           Path={
              SELECT atoi(string=Start) AS Offset,
                   atoi(string=Length) AS Length
              FROM Artifact.Windows.Sys.PhysicalMemoryRanges()
              WHERE Type = 3
           })

      -- Load the winpmem binary
      LET WinpmemBinary = SELECT FullPath
        FROM Artifact.Generic.Utils.FetchBinary(ToolName="WinPmem")

      -- Install the driver and schedule an uninstall when the query
      -- is done.  We must load the driver on the system drive or it
      -- wont load properly.
      LET _ <= SELECT *
      FROM foreach(row=WinpmemBinary,
      query={
         SELECT *, atexit(query={
            SELECT * FROM execve(argv=[FullPath, "-u"])
          }, env=dict(FullPath=FullPath)) AS AtExit
         FROM execve(argv=[FullPath, "-l"], env=dict(TMP="C:\\Windows\\Temp"))
      })

      SELECT
         Rule,
         Meta,
         String.Offset as HitOffset,
         String.Name as HitName,
         String.HexData as HitHexData
      FROM yara(files=SparsePath, accessor='sparse',
                rules=yara_rules, context=Context)

```