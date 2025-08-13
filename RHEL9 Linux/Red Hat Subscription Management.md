# Red Hat Subscription Management
You can perform the following main tasks with the Red Hat Subscription Management tools:
- _Register_ a system to associate it with the Red Hat account with an active subscription. With the Subscription Manager, the system can register uniquely in the subscription service inventory. You can unregister the system when it is not in use.
- _Subscribe_ a system to entitle it to updates for the selected Red Hat products. Subscriptions have specific levels of support, expiration dates, and default repositories. The tools help to either auto-attach or select a specific entitlement.  
- _Enable repositories_ to provide software packages. By default, each subscription enables multiple repositories; other repositories such as updates or source code are enabled or disabled. A repository is a central location for storing and maintaining software packages.
- _Review and track_ available or consumed entitlements. In the Red Hat Customer Portal, you might view the subscription information locally on a specific system or for a Red Hat account.

## Activation Keys
An _activation key_ is a preconfigured subscription management file that is available for use with both Red Hat Satellite Server and subscription management through the Red Hat Customer Portal. Use the `subscription-manager` command with activation keys to simplify the registration and assignment of predefined subscriptions.  
This method of registration is beneficial for automating installations and deployments. For organizations that enable Simple Content Access, activation keys can register systems and enable repositories without needing to attach subscriptions.  
