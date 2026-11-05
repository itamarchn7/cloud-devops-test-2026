# GCP Cloud DevOps Engineer exercise - Itamar Cohen

## Task 2 – Backup and Recovery

### 1. Required Improvements Before an Incident

The current configuration does not meet the required recovery objectives. Daily Cloud SQL backups alone are insufficient to achieve an RPO of 15 minutes.

The following improvements are recommended:

**Cloud SQL – PostgreSQL**
- Enable Point-in-Time Recovery (PITR) to allow restoration to a specific timestamp before an accidental deletion.
- Keep automated backups enabled, with at least 7 days of backup retention.
- Configure transaction log retention to cover the required recovery window.
- Perform periodic restore tests to verify that backups and PITR are usable.

**Google Cloud Storage**
- Enable Soft Delete with a retention period of at least 7 days, allowing recovery of accidentally deleted objects.
- Enable Object Versioning to retain previous object generations, including overwritten files.
- Configure lifecycle policies to manage old object generations and control storage costs.
- Periodically test object restoration.

### 2. Recovery Procedure

Assume the accidental deletion occurred at 10:20 and was discovered at 10:35.

1. **Contain the incident:** Temporarily stop application write operations by enabling maintenance mode or otherwise preventing new data changes. Preserve logs and the original database and storage state for investigation.

2. **Identify the recovery point:** Review audit logs and application logs to determine the exact deletion timestamp and the affected database records and storage objects.

3. **Restore Cloud SQL:** Use PITR to create a new Cloud SQL instance at a timestamp immediately before the accidental deletion. Restore to a separate instance rather than overwriting the original one.

4. **Restore Cloud Storage objects:** Identify the affected objects and restore them from Soft Delete. If files were overwritten, use Object Versioning to recover the required previous generations.

5. **Validate data consistency:** Verify that restored database records reference existing Cloud Storage objects. Check for missing objects, orphaned objects, and records created after the selected recovery point.

6. **Reconnect the application:** Update the application's database connection configuration to point to the restored Cloud SQL instance. Update the relevant Secret Manager configuration if required, and restart or roll out application Pods so they use the restored connection.

7. **Validate before reopening:** Test database connectivity, API operations, file access, critical application flows, and background worker functionality. Reopen the service only after the recovery checks pass.

Any valid application changes made after the selected recovery point must be identified and handled separately to avoid unnecessary data loss.

### 3. Recovery Objectives and Validation

**RPO – 15 minutes**

Enable Cloud SQL PITR and retain transaction logs so that the database can be restored to a point immediately before the incident. Validate this through periodic recovery exercises using known test records and timestamps. Confirm that the available recovery point is no more than 15 minutes behind the incident time.

**RTO – 2 hours**

Perform end-to-end recovery exercises and measure the time from incident detection until the application is fully operational. The recovery procedure must include database restoration, object recovery, application reconnection, and validation. Recovery times must be measured using production-like data volumes to demonstrate that the two-hour target is achievable.

**Deleted file recovery – 7 days**

Configure GCS Soft Delete for at least seven days and periodically test the restoration of deleted objects at different points within the retention period.

Recovery test results, timestamps, validation results, and total recovery duration should be documented as evidence that the objectives are met.

### Assumptions and Considerations

- Cloud SQL PITR and GCS Soft Delete protect against accidental data deletion but do not replace High Availability.
- Restoring the database and restoring storage objects are separate operations and must be coordinated to maintain application consistency.
- Restoring Cloud SQL to a new instance preserves the original instance for investigation and comparison.
- The two-hour RTO is a target that must be validated through realistic recovery tests; it cannot be guaranteed by configuration alone.