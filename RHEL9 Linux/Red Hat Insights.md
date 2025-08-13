# Red Hat Insights
Red Hat Insights is a predictive analytics tool to help you to identify and remediate threats to security, performance, availability, and stability on systems in your infrastructure that run Red Hat products. Red Hat delivers Red Hat Insights as a Software-as-a-Service (SaaS) product, so that you can deploy and scale it with no additional infrastructure requirements. In addition, you can take advantage of the latest recommendations and updates from Red Hat that apply to your deployed systems.
## Register a RHEL System with Red Hat Insights
Registering a RHEL server to Red Hat Insights is a quick task.
Interactively register the system with the Red Hat Subscription Management service.
[root@host ~]# **`subscription-manager register --auto-attach`**  

Ensure that the `insights-client` package is installed on your system. The package is installed by default on RHEL 8 and later systems.
[root@host ~]# **`dnf install insights-client`**  

Use the `insights-client --register` command to register the system with the Insights service and to upload initial system metadata.
[root@host ~]# **`insights-client --register`**  

On Red Hat Insights (`https://console.redhat.com/insights`), ensure that you are logged in and that the system is visible under the Inventory section of the web UI.