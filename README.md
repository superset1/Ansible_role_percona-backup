Percona Backup for MongoDB <!-- omit in toc -->
=========

This role makes MongoDB Logical, Physical, Selective, Incremental and Snapshot-based backups. Percona allows Point-in-time recovery.
See the [official page](https://docs.percona.com/percona-backup-mongodb/index.html) for more details.

Content <!-- omit in toc -->
------------
- [Supported MongoDB Versions](#supported-mongodb-versions)
- [Make a Backup](#make-a-backup)
- [Restore a Backup](#restore-a-backup)
- [Other Commands](#other-commands)
- [Notes and Problems](#notes-and-problems)

## Supported MongoDB Versions

More information on the [official page](https://docs.percona.com/percona-backup-mongodb/details/versions.html#supported-mongodb-versions).

| Backup Type | MongoDB Version                                                                                           |
| ----------- | :-------------------------------------------------------------------------------------------------------- |
| Logical     | >= 4.4                                                                                                    |
| Physical    | 4.4.6-8, 5.0 and higher with MongoDB Replication enabled and WiredTiger configured as the storage engine. |

## Make a Backup

1. Physical backup (for Percona Server for MongoDB only):
    ```
    pbm backup --type=physical 
    ```
2. Logical (for Percona Server for MongoDB and MongoDB Community Edition)
    ```
    pbm backup --type=logical
    ```
    Logical backup is the copying of the actual database data. A pbm-agent connects to the database, retrieves the data, and writes it to the remote backup storage.

## Restore a Backup

1. Connect to primary Mongod node in replicaset or Mongocfg node in sharded cluster
2. Disable point-in-time recovery, bacause a restore and point-in-time recovery oplog slicing are incompatible operations and cannot be run simultaneously:
    ```
    pbm config --set pitr.enabled=false
    ```
3. Stop the balancer and Mongos nodes
4. Stop the arbiter nodes manually since there’s no pbm-agent on these nodes to do that automatically
5. Make sure no writes are made to the database during restore
6. List backups:
    ```
    pbm list
    ```
7. Restore:
    ```
    pbm restore --time <timestamp> -w
    ```
8. Restart all Mongod nodes (Not Mongos!):
    ```
    ansible '!mongos_servers' -i hosts -m reboot
    ```
9. Connect to primary Mongod node in replicaset or Mongocfg node in sharded cluster
10. Resync the backup list with the storage:
    ```
    pbm config --force-resync
    ```
11. Start the balancer and start Mongos nodes
12. [Make a fresh backup](#make-a-backup) to serve as the new base for future restores.

## Other Commands

1. In MongoDB:
- `db.runCommand( { buildInfo: 1 } )` - get MongoDB version
- `db.getSiblingDB("admin").pbmConfig.findOne()` - get pbm config from MongoDB

2. In command line:
- `pbm config` - set, change or list the config
- `pbm status` - show PBM status
- `pbm logs` - show PBM logs
- `pbm list` - list backups
- `pbm backup` - make backup
- `pbm cancel-backup` - cancel backup
- `pbm describe-backup <backup-name> --with-collections` - describe backup
- `pbm restore` - restore backup
- `pbm oplog-replay` - replay oplog
- `pbm describe-restore 2022-08-15T11:14:55.683148162Z -c pbm-config.yaml` - describe restore
- `pbm delete-backup` - delete a backup
- `pbm delete-pitr` - delete PITR chunks
- `pbm cleanup` - delete backups and PITR chunks
- `pbm help` - help and additional commands

## Notes and Problems

1. A full backup is required to run PITR.
2. Percona can't backup to local filesystem.
