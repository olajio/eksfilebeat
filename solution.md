Ola, this is a classic Filebeat metadata enrichment timing issue that I've seen before. Looking at your data and the `eksfilebeat.yml`, I can pinpoint the likely cause.

**Root Cause: The `add_cloud_metadata` processor is racing against container startup**

In your config, the `add_cloud_metadata` processor is defined at the global `processors` level, which means it runs for all inputs. However, when a Filebeat pod starts up (or restarts due to node churn from Karpenter spot instances), there's a race condition where the cloud metadata service may not be immediately reachable from the container's network namespace.

Looking at your bad data, all the affected documents are missing the entire `cloud.*` field hierarchy, which confirms the metadata fetch failed entirely rather than being a partial enrichment issue.

**Key observations from your data:**

1. **Bad data** comes from node `ip-10-129-60-70.ec2.internal` running on `r7gd.16xlarge` spot instances
2. **Good data** comes from node `ip-10-129-164-225.ec2.internal` running on `m8gd.12xlarge` spot instances
3. Both are Karpenter-managed spot instances, meaning high node turnover
4. The timing correlates with Friday's release, which likely triggered pod restarts across nodes

**The fix - add retry logic and timeout to the cloud metadata processor:**

```yaml
processors:
  - add_cloud_metadata:
      providers: ["aws"]
      timeout: 10s
      overwrite: true
  - add_host_metadata:
  - drop_event:
      when:
        contains:
          kubernetes.container.name: "calico-node"
```

**Additional recommendations:**

1. **Add an init container or startup probe** to ensure the EC2 metadata service is reachable before Filebeat starts processing:

```yaml
initContainers:
- name: wait-for-metadata
  image: busybox:1.36
  command: ['sh', '-c', 'until wget -q -O- http://169.254.169.254/latest/meta-data/instance-id; do echo "Waiting for metadata service..."; sleep 2; done']
```

2. **Consider moving cloud metadata enrichment to an ingest pipeline** in Elasticsearch instead of relying on Filebeat. This way, if the initial enrichment fails, you can re-enrich via an update_by_query later:

```json
{
  "enrich_cloud_metadata": {
    "script": {
      "source": """
        if (ctx.cloud?.account?.id == null && ctx.host?.name != null) {
          // Flag for later enrichment
          ctx._needs_cloud_enrichment = true
        }
      """
    }
  }
}
```

3. **Check if IMDSv2 hop limit is causing issues** on your Karpenter nodes. Containers sometimes can't reach the metadata service if the hop limit is set to 1. Verify your EC2NodeClass has:

```yaml
metadataOptions:
  httpTokens: required
  httpPutResponseHopLimit: 2  # Important for containerized workloads
```

Would you like me to help you draft the complete updated `eksfilebeat.yml` with these fixes, or create a watcher to alert on documents missing cloud metadata so you can track this issue going forward?
