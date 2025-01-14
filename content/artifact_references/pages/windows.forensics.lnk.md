---
title: Windows.Forensics.Lnk
hidden: true
tags: [Client Artifact]
---

A Lnk file parser.

Status: Experimental

NOTE: Not all shell bags are currently supported, just the common ones.


```yaml
name: Windows.Forensics.Lnk
description: |
  A Lnk file parser.

  Status: Experimental

  NOTE: Not all shell bags are currently supported, just the common ones.

reference:
  - https://github.com/libyal/libfwsi/blob/main/documentation/Windows%20Shell%20Item%20format.asciidoc
  - https://github.com/libyal/liblnk/blob/main/documentation/Windows%20Shortcut%20File%20(LNK)%20format.asciidoc
  - https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-shllink


parameters:
  - name: Glob
    default: C:\Users\*\AppData\Roaming\Microsoft\Windows\Recent\**\*.lnk

  - name: AlsoUpload
    description: Also upload the link files themselves.
    type: bool

export: |
     LET Profile = '''
     [
      ["ShellLinkHeader", 0, [
        ["HeaderSize", 0, "uint32"],
        ["__LinkClsID", 4, "String", {
            "length": 16,
            "term": ""
        }],
        ["LinkClsID", 0, "Value", {
            "value": "x=>format(format='%x', args=x.__LinkClsID)"
        }],
        ["LinkFlags", 20, "Flags", {
            "type": "uint32",
            "bitmap": {
                "HasLinkTargetIDList": 0,
                "HasLinkInfo": 1,
                "HasName": 2,
                "HasRelativePath": 3,
                "HasWorkingDir": 4,
                "HasArguments": 5,
                "HasIconLocation": 6,
                "IsUnicode": 7,
                "ForceNoLinkInfo": 8,
                "HasExpString": 9,
                "RunInSeparateProcess": 10,
                "HasDarwinID": 12,
                "RunAsUser": 13,
                "HasExpIcon": 14,
                "NoPidlAlias": 15,
                "RunWithShimLayer": 17,
                "ForceNoLinkTrack": 18,
                "EnableTargetMetadata": 19,
                "DisableLinkPathTracking": 20,
                "DisableKnownFolderTracking": 21,
                "DisableKnownFolderAlias": 22,
                "AllowLinkToLink": 23,
                "UnaliasOnSave": 24,
                "PreferEnvironmentPath": 25,
                "KeepLocalIDListForUNCTarget": 26
            }
        }],
        ["FileAttributes", 24, "Flags", {
            "type": "uint32",
            "bitmap": {
                "FILE_ATTRIBUTE_READONLY": 0,
                "FILE_ATTRIBUTE_HIDDEN": 1,
                "FILE_ATTRIBUTE_SYSTEM": 2,
                "FILE_ATTRIBUTE_DIRECTORY": 4,
                "FILE_ATTRIBUTE_ARCHIVE": 5,
                "FILE_ATTRIBUTE_NORMAL": 7,
                "FILE_ATTRIBUTE_TEMPORARY": 8,
                "FILE_ATTRIBUTE_SPARSE_FILE": 9,
                "FILE_ATTRIBUTE_REPARSE_POINT": 10,
                "FILE_ATTRIBUTE_COMPRESSED": 11,
                "FILE_ATTRIBUTE_OFFLINE": 12,
                "FILE_ATTRIBUTE_NOT_CONTENT_INDEXED": 13,
                "FILE_ATTRIBUTE_ENCRYPTED": 14,
            }
        }],
        ["CreationTime", 28, "WinFileTime", {
            "type": "uint64"
        }],
        ["AccessTime", 36, "WinFileTime", {
            "type": "uint64"
        }],
        ["WriteTime", 44, "WinFileTime", {
            "type": "uint64"
        }],
        ["FileSize", 52, "uint32"],
        ["IconIndex", 56, "uint32"],
        ["ShowCommand", 60, "uint32"],
        ["HotKey", 62, "uint16"],

        # The LinkTargetIDList only exists if the Link Flag is set otherwise it is empty.
        ["LinkTargetIDList", "x=>x.HeaderSize", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasLinkTargetIDList'",
            "choices": {
                "true": "LinkTargetIDList",
                "false": "Empty"
            }
         }],

        ["LinkInfo", "x=>x.LinkTargetIDList.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasLinkInfo'",
            "choices": {
                "true": "LinkInfo",
                "false": "Empty"
            }
        }],

        ["NameInfo", "x=>x.LinkInfo.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasName'",
            "choices": {
                "true": "NameInfo",
                "false": "Empty"
            }
        }],

        ["RelativePathInfo", "x=>x.NameInfo.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasRelativePath'",
            "choices": {
                "true": "RelativePathInfo",
                "false": "Empty"
            }
        }],

        ["WorkingDirInfo", "x=>x.RelativePathInfo.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasWorkingDir'",
            "choices": {
                "true": "WorkingDirInfo",
                "false": "Empty"
            }
        }],

        ["ArgumentInfo", "x=>x.WorkingDirInfo.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasArguments'",
            "choices": {
                "true": "ArgumentInfo",
                "false": "Empty"
            }
        }],
        ["IconInfo", "x=>x.ArgumentInfo.EndOf", "Union", {
            "selector": "x=>x.LinkFlags =~ 'HasIconLocation'",
            "choices": {
                "true": "IconInfo",
                "false": "Empty"
            }
        }]
      ]],
      ["Empty", 0, []],

      # Struct size includes the size field
      ["LinkTargetIDList", "x=>x.IDListSize + 2", [
        ["IDListSize", 0, "uint16"],
        ["IDList", 2, "Array", {
           "type": "ItemIDList",
           "count": 1000   # Max count until sentinal
         }]
      ]],

      # Item List contains shell bags
      ["ItemIDList", "x=>x.ItemIDSize", [
        ["ItemIDSize", 0, "uint16"],
        ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
        ["Type", 2, "BitField", {
          "type": "uint8",
          "start_bit": 4,
          "end_bit": 7,
        }],

        ["Subtype", 2, "BitField", {
           "type": "uint8",
           "start_bit": 0,
           "end_bit": 1,
        }],

        # For now only support some common shell bags
        ["ShellBag", 0, "Union", {
           "selector": "x=>x.Type",
            "choices": {
               "64": "ShellBag0x40",
               "48": "ShellBag0x30",
               "16": "ShellBag0x1f",
               "32": "ShellBag0x20",
            }
        }]
      ]],

      ["ShellBag0x40", 0, [
         ["Name", 5, "String", {
            encoding: "utf8",
         }],
         ["Description", 0, "Value", {
             "value": 'x=>dict(
             Type="NetworkLocation",
             ShortName=x.Name
             )'
         }]
      ]],

      # A LinkInfo stores information about the destination of the link.
      ["LinkInfo", "x=>x.LinkInfoSize", [
        ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
        ["LinkInfoSize", 0, "uint32"],
        ["__LinkInfoHeaderSize", 4, "uint32"],
        ["LinkInfoFlags", 8, "Flags", {
            "type": "uint32",
            "bitmap": {
                "VolumeIDAndLocalBasePath": 0,
                "CommonNetworkRelativeLinkAndPathSuffix": 1
            }
        }],
        ["__VolumeIDOffset", 0xc, "uint32"],
        ["__LocalBasePathOffset", 16, "uint32"],
        ["__CommonNetworkRelativeLinkOffset", 20, "uint32"],
        ["__CommonPathSuffixOffset", 24, "uint32"],
        ["__LocalBasePath", "x=>x.__LocalBasePathOffset", "String", {}],
        ["__CommonNetworkRelativePath", "x=>x.__CommonNetworkRelativeLinkOffset", "String"],
        ["__CommonPathSuffix", "x=>x.__CommonPathSuffixOffset", "String"],
        ["__VolumeID", "x=>x.__VolumeIDOffset", "VolumeID"],
        ["__CommonNetworkRelativeLink", "x=>x.__CommonNetworkRelativeLinkOffset", "CommonNetworkRelativeLink"],

        # Depending on the LinkInfoFlags this struct needs to be interpreted differently.
        ["Target", 0, "Value", {
            "value": '
               x=>if(condition=x.LinkInfoFlags =~ "VolumeIDAndLocalBasePath",
                     then=dict(path=x.__LocalBasePath,
                               volume_info=x.__VolumeID),
                     else=dict(path=format(format="%v\\%v",
                               args=[x.__CommonNetworkRelativeLink.NetName, x.__CommonPathSuffix]),
                               relative_link=x.__CommonNetworkRelativeLink)
             )'
        }]
      ]],

      ["CommonNetworkRelativeLink", 0, [
        ["__CommonNetworkRelativeLinkSize", 0, "uint32"],
        ["__CommonNetworkRelativeLinkFlags", 4, "Flags", {
            "type": "uint32",
            "bitmap": {
                "ValidDevice": 0,
                "ValidNetType": 1,
            }
        }],
        ["__NetNameOffset", 8, "uint32"],
        ["__DeviceNameOffset", 12, "uint32"],
        ["NetworkProviderType", 16, "Enumeration", {
            "type": "uint32",
            "map": {
                "WNNC_NET_AVID": 0x001A0000,
                "WNNC_NET_DOCUSPACE": 0x001B0000,
                "WNNC_NET_MANGOSOFT": 0x001C0000,
                "WNNC_NET_SERNET": 0x001D0000,
                "WNNC_NET_RIVERFRONT1": 0X001E0000,
                "WNNC_NET_RIVERFRONT2": 0x001F0000,
                "WNNC_NET_DECORB": 0x00200000,
                "WNNC_NET_PROTSTOR": 0x00210000,
                "WNNC_NET_FJ_REDIR": 0x00220000,
                "WNNC_NET_DISTINCT": 0x00230000,
                "WNNC_NET_TWINS": 0x00240000,
                "WNNC_NET_RDR2SAMPLE": 0x00250000,
                "WNNC_NET_CSC": 0x00260000,
                "WNNC_NET_3IN1": 0x00270000,
                "WNNC_NET_EXTENDNET": 0x00290000,
                "WNNC_NET_STAC": 0x002A0000,
                "WNNC_NET_FOXBAT": 0x002B0000,
                "WNNC_NET_YAHOO": 0x002C0000,
                "WNNC_NET_EXIFS": 0x002D0000,
                "WNNC_NET_DAV": 0x002E0000,
                "WNNC_NET_KNOWARE": 0x002F0000,
                "WNNC_NET_OBJECT_DIRE": 0x00300000,
                "WNNC_NET_MASFAX": 0x00310000,
                "WNNC_NET_HOB_NFS": 0x00320000,
                "WNNC_NET_SHIVA": 0x00330000,
                "WNNC_NET_IBMAL": 0x00340000,
                "WNNC_NET_LOCK": 0x00350000,
                "WNNC_NET_TERMSRV": 0x00360000,
                "WNNC_NET_SRT": 0x00370000,
                "WNNC_NET_QUINCY": 0x00380000,
                "WNNC_NET_OPENAFS": 0x00390000,
                "WNNC_NET_AVID1": 0X003A0000,
                "WNNC_NET_DFS": 0x003B0000,
                "WNNC_NET_KWNP": 0x003C0000,
                "WNNC_NET_ZENWORKS": 0x003D0000,
                "WNNC_NET_DRIVEONWEB": 0x003E0000,
                "WNNC_NET_VMWARE": 0x003F0000,
                "WNNC_NET_RSFX": 0x00400000,
                "WNNC_NET_MFILES": 0x00410000,
                "WNNC_NET_MS_NFS": 0x00420000,
                "WNNC_NET_GOOGLE": 0x00430000,
            }
        }],
        ["__NetNameOffsetUnicode", 20, "uint32"],
        ["__DeviceNameOffsetUnicode", 24, "uint32"],
        ["__NetNameAscii", "x=>x.__NetNameOffset", "String"],
        ["__DeviceNameAscii", "x=>x.__DeviceNameOffset", "String"],
        ["__NetNameUnicode", "x=>x.__NetNameOffsetUnicode", "String", {"encoding": "utf16"}],
        ["__DeviceNameUnicode", "x=>x.__DeviceNameOffsetUnicode", "String", {"encoding": "utf16"}],
        ["NetName", 0, "Value", {
            "value": "x=>if(condition=x.__NetNameOffset, then=x.__NetNameAscii, else=x.__NetNameUnicode)"
        }],
        ["DeviceName", 0, "Value", {
            "value": "x=>if(condition=x.__DeviceNameOffset, then=x.__DeviceNameAscii, else=x.__DeviceNameUnicode)"
        }]
      ]],

      # This is a comment
      ["VolumeID", 0, [
        ["__VolumeIDSize", 0, "uint32"],
        ["DriveType", 4, "Enumeration", {
            "type": "uint32",
            "choices": {
                 "0": "DRIVE_UNKNOWN",
                 "1": "DRIVE_NO_ROOT_DIR",
                 "2": "DRIVE_REMOVABLE",
                 "3": "DRIVE_FIXED",
                 "4": "DRIVE_REMOTE",
                 "5": "DRIVE_CDROM",
                 "6": "DRIVE_RAMDISK"
            }
        }],
        ["DriveSerialNumber", 8, "uint32"],
        ["__VolumeLabelOffset", 12, "uint32"],
        ["__VolumeLabelOffsetUnicode", 16, "uint32"],
        ["__VolumeLabelAscii", "x=>x.__VolumeLabelOffset", "String"],
        ["__VolumeLabelUnicode", "x=>x.__VolumeLabelOffsetUnicode", "String", {"encoding": "utf16"}],
        ["VolumeLabel", 0, "Value", {
            "value": 'x=>if(condition=x.__VolumeLabelOffset,
               then=x.__VolumeLabelAscii, else=x.__VolumeLabelUnicode)'
        }]
      ]],

      # Volume name
      ["ShellBag0x20", 0, [
         ["__Name", 3, "String"],
         # Name is only valid if the first bit is set.
         ["Name", 3, "Value", {
             "value": "x=>if(condition=x.ParentOf.Subtype, then=x.__Name, else='')",
         }],
         ["Description", 0, "Value", {
            "value": 'x=>dict(
                LongName=x.Name,
                ShortName=x.Name,
                Type="Volume"
            )'
        }]
      ]],

      # Marks the root class My Computer
      ["ShellBag0x1f", 0, [
        ["Description", 0, "Value", {
            "value": 'x=>dict(
               ShortName="My Computer",
               Type="Root"
            )'
        }]
      ]],

      # Represent a file or directory
      ["ShellBag0x30", 0, [
        ["Size", 0, "uint16"],
        ["Type", 2, "uint8"],
        ["SubType", 2, "Flags", {
            "type": "uint8",
            "bitmap": {
                "File": 1,
                "Directory": 0,
                "Unicode": 4,
            }
        }],
        ["__LastModificationTime", 8, "uint32"],
        ["LastModificationTime", 8, "FatTimestamp"],
        ["ShortName", 14, "String"],

        # Variable length search for the extension signature from the start of the struct.
        ["__pre", 0, "String", {
            "term_hex": "0400efbe"
        }],

        # The extension tag should be immediately after the search string.
        ["__ExtensionTag", "x=>len(list=x.__pre)", "uint32"],

        # Extension starts 4 bytes before the tag
        ["Extension", "x=>len(list=x.__pre) - 4", "Union", {
            "selector": "x=>format(format='%#x', args=x.__ExtensionTag)",
            "choices": {
               "0xbeef0004": "Beef0004",
            }
         }],

         # Put all the data together in a convenient location
         ["Description", 0, "Value", {
             "value": 'x=>dict(
                 Type=x.SubType,
                 Modified=if(condition=x.__LastModificationTime, then=x.LastModificationTime),
                 LastAccessed=if(condition=x.Extension.__LastAccessed, then=x.Extension.LastAccessed),
                 CreateDate=if(condition=x.Extension.__CreateDate, then=x.Extension.CreateDate),
                 ShortName=x.ShortName,
                 LongName=x.Extension.LongName,
                 MFTID=x.Extension.MFTReference.MFTID,
                 MFTSeq=x.Extension.MFTReference.SequenceNumber
             )'
         }]
       ]],
       ["Beef0004", 0, [
          ["Size", 0, "uint16"],
          ["Version", 2, "uint16"],
          ["__Signature", 4, "uint32"],
          ["Signature", 0, "Value", {
              "value": "x=>format(format='%#x', args=x.__Signature)"
          }],
          ["__CreateDate", 8, "uint32"],
          ["__LastAccessed", 12, "uint32"],

          ["CreateDate", 8, "FatTimestamp"],
          ["LastAccessed", 12, "FatTimestamp"],
          ["MFTReference", 20, "MFTReference"],
          ["LongName", 46, "String", {
              "encoding": "utf16"
          }]
       ]],
       ["MFTReference", 0, [
         ["MFTID", 0, "BitField", {
             "type": "uint64",
             "start_bit": 0,
             "end_bit": 48,
         }],
         ["SequenceNumber", 0, "BitField", {
             "type": "uint64",
             "start_bit": 48,
             "end_bit": 64,
         }]
       ]],

       ["NameInfo", "x=>x.NameInfoSize + 2", [
            ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
            ["NameInfoSize", 0, "Value", {
                "value": "x=>x.__NameInfoSize * 2"
            }],
            ["Name", 2, "String", {
                "encoding": "utf16",
                "length": "x=>x.NameInfoSize",
            }],
            ["__NameInfoSize", 0, "uint16"],
       ]],

       ["WorkingDirInfo", "x=>x.WorkingDirInfoSize + 2", [
            ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
            ["WorkingDirInfoSize", 0, "Value", {
                "value": "x=>x.__WorkingDirInfoSize * 2"
            }],
            ["WorkingDir", 2, "String", {
                "encoding": "utf16",
                "length": "x=>x.WorkingDirInfoSize",
            }],
            ["__WorkingDirInfoSize", 0, "uint16"],
       ]],

       ["RelativePathInfo", "x=>x.RelativePathInfoSize + 2", [
            ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
            ["RelativePathInfoSize", 0, "Value", {
                "value": "x=>x.__RelativePathInfoSize * 2"
            }],
            ["RelativePath", 2, "String", {
                "encoding": "utf16",
                "length": "x=>x.RelativePathInfoSize",
            }],
            ["__RelativePathInfoSize", 0, "uint16"],
       ]],

       ["ArgumentInfo", "x=>x.ArgumentInfoSize + 2", [
            ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
            ["ArgumentInfoSize", 0, "Value", {
                "value": "x=>x.__ArgumentInfoSize * 2"
            }],
            ["Arguments", 2, "String", {
                "encoding": "utf16",
                "length": "x=>x.ArgumentInfoSize",
            }],
            ["__ArgumentInfoSize", 0, "uint16"],
       ]],
       ["IconInfo", "x=>x.IconInfoSize + 2", [
            ["Offset", 0, "Value", {"value": "x=>x.StartOf"}],
            ["IconInfoSize", 0, "Value", {
                "value": "x=>x.__IconInfoSize * 2"
            }],
            ["IconLocations", 2, "String", {
                "encoding": "utf16",
                "length": "x=>x.IconInfoSize",
            }],
            ["__IconInfoSize", 0, "uint16"],
       ]]
     ]
     '''

sources:
  - query: |
     LET link_files = SELECT parse_binary(
         filename=FullPath,
         profile=Profile, struct="ShellLinkHeader")  AS Parsed,
         FullPath, Mtime as SourceModified, Btime as SourceCreated
     FROM glob(globs=Glob)

     SELECT FullPath, Parsed AS _Parsed, SourceCreated, SourceModified,
            Parsed.LinkTargetIDList.IDList.ShellBag.Description AS _TargetIDInfo,
            Parsed.CreationTime AS HeaderCreationTime,
            Parsed.AccessTime AS HeaderAccessTime,
            Parsed.WriteTime AS HeaderWriteTime,
            Parsed.FileSize AS FileSize,
            Parsed.LinkInfo.Target AS Target,
            Parsed.NameInfo.Name AS Name,
            Parsed.RelativePathInfo.RelativePath AS RelativePath,
            Parsed.WorkingDirInfo.WorkingDir AS WorkingDir,
            Parsed.ArgumentInfo.Arguments AS Arguments,
            Parsed.IconInfo.IconLocations AS Icons,
            if(condition=AlsoUpload, then=upload(file=FullPath)) AS Upload
     FROM link_files

column_types:
  - name: Mtime
    type: timestamp
  - name: Atime
    type: timestamp
  - name: Btime
    type: timestamp
  - name: HeaderCreationTime
    type: timestamp
  - name: HeaderAccessTime
    type: timestamp
  - name: HeaderWriteTime
    type: timestamp

```
