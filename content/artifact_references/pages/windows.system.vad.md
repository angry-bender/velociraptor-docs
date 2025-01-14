---
title: Windows.System.VAD
hidden: true
tags: [Client Artifact]
---

This artifact enables enumeration of process memory sections via the Virtual 
Address Descriptor (VAD). The VAD is used by the Windows memory manager to 
describe allocated process memory ranges.

Availible filters include process, mapping path, memory permissions 
or by content with yara.

NOTE:  ProtectionChoice is a choice to filter on section protection. Default is 
all sections and ProtectionRegex can override selection.   
To filter on unmapped sections the MappingNameRegex: ^$ can be used.


```yaml
name: Windows.System.VAD
author: "Matt Green - @mgreen27"
description: |
  This artifact enables enumeration of process memory sections via the Virtual 
  Address Descriptor (VAD). The VAD is used by the Windows memory manager to 
  describe allocated process memory ranges.
  
  Availible filters include process, mapping path, memory permissions 
  or by content with yara.
  
  NOTE:  ProtectionChoice is a choice to filter on section protection. Default is 
  all sections and ProtectionRegex can override selection.   
  To filter on unmapped sections the MappingNameRegex: ^$ can be used.

parameters:
  - name: ProcessRegex
    description: A regex applied to process names.
    default: .
    type: regex
  - name: PidRegex
    default: .
    type: regex
  - name: ProtectionChoice
    type: choices
    description: Select memory permission you would like to return. Default All.
    default: Any
    choices:
      - Any
      - Execute, read and write
      - Any executable
  - name: ProtectionRegex
    type: regex
    description: Allows a manual regex selection of section Protection permissions. If configured take preference over Protection choice. 
  - name: MappingNameRegex
    type: regex
  - name: UploadSection
    description: Upload suspicious section.
    type: bool
  - name: SuspiciousContent
    description: A yara rule of suspicious section content 
    type: yara
  - name: ContextBytes
    description: Include this amount of bytes around yara hit as context.
    default: 10000
    type: int
    
    
sources:
  - query: |
      -- firstly find processes in scope
      LET processes = SELECT Pid, Name,Exe,CommandLine,CreateTime
        FROM pslist()
        WHERE Name =~ ProcessRegex
            AND format(format="%d", args=Pid) =~ PidRegex
            AND log(message="Scanning pid %v : %v", args=[Pid, Name])

      -- next find sections in scope
      LET sections <= SELECT * FROM foreach(
          row=processes,
          query={
            SELECT CreateTime as ProcessCreateTime,Pid, Name,MappingName  ,
                format(format='%x-%x', args=[Address, Address+Size]) AS AddressRange,
                Protection, Address as _Address,
                Size as SectionSize,
                pathspec(
                    DelegateAccessor="process",
                    DelegatePath=Pid,
                    Path=Address) AS _PathSpec
            FROM vad(pid=Pid)
            WHERE if(condition=MappingNameRegex,
                    then= MappingName=~MappingNameRegex,
                    else= True)
                AND if(condition = ProtectionRegex,
                    then= Protection=~ProtectionRegex,
                    else= if(condition= ProtectionChoice='Any',
                        then= TRUE,
                    else= if(condition= ProtectionChoice='Execute, read and write',
                        then= Protection= 'xrw',
                    else= if(condition= ProtectionChoice='Any executable',
                        then= Protection=~'x'))))
          })
      
      -- if suspicious yara added, search for it
      LET yara_sections = SELECT *
        FROM foreach(row=sections,
            query={
                SELECT
                    ProcessCreateTime, Pid, Name,MappingName,
                    AddressRange,Protection,SectionSize,
                    enumerate(items=dict(
                        Rule=Rule,
                        Meta=Meta,
                      Tags=Tags,
                      String=String))[0] as YaraHit,
                    _PathSpec,_Address
                FROM yara(
                            accessor='offset',
                            files=_PathSpec, 
                            rules=SuspiciousContent,
                            end=SectionSize,  key='X', 
                            number=1,
                            context=ContextBytes
                        )
            })
      
      -- finalise results
      LET results = SELECT *, process_tracker_callchain(id=Pid).Data as ProcessChain
        FROM if(condition= SuspiciousContent,
                    then= yara_sections,
                    else= sections) 
      
      -- upload sections if selected        
      LET upload_results = SELECT *, 
                                upload(accessor='sparse', 
                                file=pathspec(
                                DelegateAccessor="process",
                                DelegatePath=Pid,
                                Path=[dict(Offset=_Address, Length=SectionSize),]), 
                                name=format(format='%v-%v_%v.bin',args= [ Name, Pid, AddressRange ])
                            ) as SectionDump
            FROM results
      
      -- output rows
      SELECT * FROM if(condition= UploadSection,
                    then= upload_results,
                    else= results)

```
