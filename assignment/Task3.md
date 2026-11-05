# GCP Cloud DevOps Engineer exercise - Itamar Cohen

## Task 3 – Linux Build Agent Disk Full

### 1. Problem Analysis

The self-hosted Ubuntu Azure DevOps Build Agent has run out of disk space.

According to the provided diagnostics:

| Location | Disk Usage |
|---|---:|
| Root filesystem (`/`) | 100 GB total, 99 GB used |
| `/var/lib/docker` | 76 GB |
| `/var/lib/other` | 4 GB |
| `/opt/ado-agent/_work` | 12 GB |

The main contributors are Docker data and the Azure DevOps working directory.
A full disk may cause Docker builds and image pulls to fail, Git checkout operations to fail, temporary files and logs not to be written, and Azure DevOps pipeline jobs to fail.
The agent is a CI/CD build machine, not an application host. Therefore, the recovery procedure should focus on restoring the build environment without deleting active pipeline data or important Docker volumes.

### 2. Investigation and Diagnostics

Before deleting anything, I would identify the exact source of disk consumption.

**Filesystem usage:**

```bash
df -h
df -hT
df -ih
```

These commands show filesystem capacity, usage, available space, filesystem type, and inode consumption.

**Directory usage:**

```bash
sudo du -xhd1 /
sudo du -xhd1 /var
sudo du -xhd1 /var/lib
sudo du -xhd1 /opt/ado-agent
```

This identifies the largest directories without crossing filesystem boundaries.

**Docker investigation:**

```bash
docker system df
docker ps -a --size
docker images
```

These commands identify disk consumption by images, containers, volumes, and build cache.

**Azure DevOps working directory:**

```bash
sudo du -sh /opt/ado-agent/_work/*
```

The `_work` directory may contain old source checkouts, build outputs, artifacts, and temporary files from previous pipeline runs.

**Logs and deleted open files:**

```bash
sudo journalctl --disk-usage
sudo du -sh /var/log
sudo lsof +L1
```

The last command helps identify deleted files that are still held open by running processes and therefore continue consuming disk space.

### 3. Immediate Recovery – Safe Cleanup

Before cleanup, I would prevent new pipeline jobs from being assigned to the affected agent and verify that no active job is using the directories or resources being cleaned.

**Docker cleanup:**

First, remove unused dangling images:

```bash
docker image prune
```

Then remove build cache older than seven days:

```bash
docker builder prune --filter "until=168h"
```

If required, remove stopped containers after verifying that they are no longer needed:

```bash
docker container prune
```

If disk pressure remains, unused images can also be removed after checking that required images are available in Artifact Registry:

```bash
docker image prune -a
```

This may require images to be downloaded again during future builds.

I would not blindly run `docker system prune -a --volumes`, because unused volumes may still contain important data or caches. I would also never manually delete `/var/lib/docker`.

**Azure DevOps workspace cleanup:**

I would inspect the `_work` directory and identify old workspaces that are not being used by active jobs.

```bash
sudo du -sh /opt/ado-agent/_work/*
sudo find /opt/ado-agent/_work -mindepth 1 -maxdepth 1 -type d -mtime +14 -print
```

The `find` command is for investigation only. Before removing any workspace, I would stop or safely drain the agent, verify that no pipeline is using it, and confirm that its contents can be recreated.

I would use Azure DevOps workspace maintenance and pipeline workspace cleanup where appropriate, rather than deleting agent directories indiscriminately. I would not routinely delete `_tasks` or `_tool`, since these directories contain agent tasks and installed tools.

**Log cleanup:**

If system journal logs are identified as a significant contributor, old logs can be removed according to the organization's retention policy:

```bash
sudo journalctl --vacuum-time=7d
```

All cleanup actions should be followed by another disk usage check.

### 4. Agent Recovery and Validation

After freeing disk space, I would verify the filesystem and Docker status:

```bash
df -h
df -ih
docker info
docker system df
docker ps -a
```

Then I would check the Azure DevOps agent service:

```bash
cd /opt/ado-agent
sudo ./svc.sh status
```

If the service is stopped and the underlying issue is resolved:

```bash
sudo ./svc.sh start
```

Finally, I would run a validation pipeline that performs:

1. Git checkout.
2. Application build.
3. Docker image build.
4. Image push to Artifact Registry.
5. Successful pipeline completion.

This confirms that the agent can perform its normal CI/CD responsibilities again.

### 5. Preventive Measures

To prevent recurrence, I would implement the following measures:

**Automated cleanup**
- Periodically remove unused Docker build cache and stopped containers.
- Remove unused local images when they can be pulled again from Artifact Registry.
- Clean old Azure DevOps workspaces using supported maintenance mechanisms.
- Apply an appropriate system log retention policy.

**Docker log rotation**

Configure Docker log rotation in `/etc/docker/daemon.json`, while preserving any existing configuration:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

This limits individual container log files and prevents unlimited log growth.

**Disk monitoring and alerts**

Configure disk usage monitoring with the following suggested thresholds:

- 70% – Early warning.
- 80% – Investigate disk growth.
- 90% – Critical alert and cleanup action.
- 95% – Immediate intervention.

Inode usage should also be monitored.

Cloud Monitoring or an existing enterprise monitoring platform can be used for these alerts.

**Storage capacity**

If the agent's workload regularly requires more storage, increase the VM disk size or consider placing Docker data and Azure DevOps workspaces on separate disks. Any Docker data-root migration should be planned and performed carefully.

**Ephemeral build agents**

As a longer-term improvement, consider ephemeral build agents. A temporary agent is provisioned for a pipeline, executes the build, publishes its artifacts, and is then discarded.

This reduces the accumulation of old workspaces, Docker images, and build cache on long-lived machines.

### 6. Assumptions and Safety Considerations

- The Ubuntu VM is used as an Azure DevOps build agent, not as an application runtime host.
- Docker images required for future builds are available from a trusted registry such as Artifact Registry.
- Active pipeline jobs and required artifacts must not be deleted during cleanup.
- Docker volumes must be inspected before removal.
- Cleanup operations should be performed in a controlled manner and followed by validation.
- The proposed commands and preventive measures are recommendations; they have not been executed against the assignment's GCP environment.