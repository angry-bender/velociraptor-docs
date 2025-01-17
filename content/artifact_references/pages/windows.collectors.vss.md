---
title: Windows.Collectors.VSS
hidden: true
tags: [Client Artifact]
---

Collects files with VSS deduplication.

Volume shadow copies is a windows feature where file system
snapshots can be made at various times. When collecting files it is
useful to go back through the VSS to see older versions of critical
files.

At the same time we dont want to collect multiple copies of the
same data.

This artifact runs the provided globs over all the VSS and collects
the unique modified time + path combinations.

If a file was modified in a previous VSS copy, this artifact will
retrieve it at multiple shadow copies.


```yaml
name: Windows.Collectors.VSS
description: |
   Collects files with VSS deduplication.

   Volume shadow copies is a windows feature where file system
   snapshots can be made at various times. When collecting files it is
   useful to go back through the VSS to see older versions of critical
   files.

   At the same time we dont want to collect multiple copies of the
   same data.

   This artifact runs the provided globs over all the VSS and collects
   the unique modified time + path combinations.

   If a file was modified in a previous VSS copy, this artifact will
   retrieve it at multiple shadow copies.

parameters:
  - name: collectionSpec
    description: |
       A CSV file with a Glob column with all the globs to collect.
       NOTE: Globs must not have a leading device since the device
       will depend on the VSS.
    type: csv
    default: |
       Glob
       Users\*\NTUser.dat
  - name: RootDevice
    description: The device to apply all the glob on.
    default: "C:"
  - name: Accessor
    default: lazy_ntfs
  - name: VSSDateRegex
    default: .
    type: regex

sources:
   - name: All Matches Metadata
     query: |
      LET originating_machine <= SELECT Data.SystemName AS System
            FROM glob(globs="/*", accessor=Accessor)
            WHERE Name = "\\\\.\\" + RootDevice

      // Generate the collection globs for each device
      LET specs = SELECT Device + Glob AS Glob
            FROM collectionSpec
            WHERE log(message="Processing Device " + Device + " With " + Accessor)

      // Join all the collection rules into a single Glob plugin. This ensure we
      // only make one pass over the filesystem. We only want LFNs.
      LET hits = SELECT FullPath AS SourceFile, Size,
               Ctime AS Created,
               Mtime AS Modified,
               Atime AS LastAccessed,
               Device, strip(string=FullPath, prefix=Device) AS Path,
               Data.mft AS MFT, Data.name_type AS NameType
        FROM glob(globs=specs.Glob, accessor=Accessor)
        WHERE Mode.IsRegular

      // Get all volume shadows on this system.
      LET volume_shadows = SELECT Data.InstallDate AS InstallDate,
               Data.DeviceObject + "\\" AS Device
        FROM glob(globs='/*', accessor=Accessor)
        WHERE Device =~ 'VolumeShadowCopy' AND
              Data.OriginatingMachine =~ originating_machine.System[0] AND
              InstallDate =~ VSSDateRegex

      // The target devices are the root device and all the VSS
      LET target_devices = SELECT * FROM chain(
            a={SELECT "\\\\.\\" + RootDevice + "\\" AS Device from scope()},
            b=volume_shadows)

      // Get all the paths matching the collection globs.
      LET all_matching = SELECT * FROM foreach(row=target_devices, query=hits)

      // Create a unique key to group by - modification time and path name.
      // Order by device name so we get C:\ above the VSS device.
      LET all_results <= SELECT Created, LastAccessed, Path, MFT, NameType,
              Modified, Size, SourceFile, Device,
              format(format="%s:%v", args=[Modified, MFT]) AS Key
        FROM all_matching ORDER BY Device DESC

      SELECT * FROM all_results

   - name: Uploads
     query: |
      // Get all the unique versions of the sort key - that is unique instances of
      // mod time + path. If a path has several mod time (i.e. different times in each VSS
      // we will get them all). But if the same path has the same mod time in all VSS we only
      // take the first one which due to the sorting above will be the root device usually.
      LET unique_mtimes = SELECT * FROM all_results GROUP BY Key

      // Upload the files using the MFT accessor.
      LET uploaded_files = SELECT * FROM foreach(row=unique_mtimes,
        workers=30,
        query={
            SELECT Created, LastAccessed, Modified, MFT, SourceFile, Size,
               upload(file=Device+MFT, name=SourceFile, accessor="mft") AS Upload
            FROM scope()
        })

      // Seperate the hashes into their own column.
      SELECT now() AS CopiedOnTimestamp, SourceFile, Upload.Path AS DestinationFile,
               Size AS FileSize, Upload.sha256 AS SourceFileSha256,
               Created, Modified, LastAccessed, MFT
      FROM uploaded_files

```
