Building on top of [Prometheus Alertmanager](https://prometheus.io/docs/alerting/alertmanager/),
OpenPAI started to support sending cluster failure and issues emails to administrator since 0.7
release.

# Configuration

To enable Alert Manager, please configure Alert Manager by adding `alerting` fields under `prometheus`
to services-configuration file.

Refer to example [`cluster-configuration`](../../cluster-configuration/cluster-configuration.yaml) and
[`service-configuration`](../../cluster-configuration/services-configuration.yaml) for more
information.

`alerting` fields has following subfield:

| Field Name | Description |
| --- | --- |
| alert_manager_port | port for alert manager to listen, make sure this port is not used in node |
| alert_receiver | which email should receive alert email |
| smtp_url | smtp server url for alert manager to connect |
| smtp_from | this email address is where alerting email sent from |
| smtp_auth_username | use this user name to login to smtp server. This user should be able to send email as `smtp_from`, can be same with `smtp_from` |
| smtp_auth_password | use this password to login to smtp server |

More advanced configurations for alert manager is not supported in pai, see official alert manager
[document](https://prometheus.io/docs/alerting/configuration/) for more options.

# Alerting rule

To facilitate the OpenPAI usage, we had predefined few alerting rules for OpenPAI.
Checkout [rule directory](../prometheus-alert) to see rules we defined.

Following are these rule's triggering condition:

| Rule name | Triggering condition |
| --- | --- |
| k8sApiServerNotOk | response from api server's healthz page is not ok or connection error |
| k8sEtcdNotOk | response from etcd server's healthz page is not ok or connection error |
| k8sKubeletNotOk | response from each kubelet's healthz page is not ok or connection error |
| k8sDockerDaemonNotOk | docker daemon's status is not ok |
| NodeFilesystemUsage | One of node's free space in file system is less than 20% |
| NodeMemoryUsage | One of node's free memory is less than 20% |
| NodeCPUUsage | One of node's cpu idle time is less than 20% |
| NodeDiskPressure | kubernetes indicate one of node is under disk pressure |
| PaiServicePodNotRunning | kubernetes indicate one of pai service pod is not in running status |
| PaiServicePodNotReady | kubernetes indicate one of pai service pod is not in ready status |

Our email template is similar to original Alert Manager's, except We only render annotation.summary
if the key exist. This can make alert email simpler to read and understand.

If you want to add more rules, please reference syntax
[here](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).
After adding rules, you should stop and start prometheus by using paictl

```
cd $pai-management
./paictl.py service stop -p $pai-config -n prometheus
./paictl.py service start -p $pai-config -n prometheus
```

Please fire a pull request if you find any rule useful.
