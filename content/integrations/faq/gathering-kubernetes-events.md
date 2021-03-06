---
title: Gathering Kubernetes events
kind: faq
---

As of the 5.17.0 release, the Datadog Agent supports built in leader election for the Kubernetes event collector.

Agents coordinate by performing a leader election among members of the Datadog DaemonSet through kubernetes to ensure only one leader agent instance is gathering events at a given time. If the leader agent instance fails, a re-election occurs and another cluster agent will take over collection.

In your `kubernetes.yaml` file you will see the [leader_lease_duration](https://github.com/DataDog/integrations-core/blob/master/kubernetes/conf.yaml.example#L118) parameter. It's the duration for which a leader stays elected. **It should be > 30 seconds**.

A longer lease duration will result in fewer agent requests to the apiserver, but if the leader dies under certain conditions, an event blackout may occur until the lease expires and a new leader is elected.

This feature relies on [ConfigMaps](https://kubernetes.io/docs/api-reference/v1.7/#configmap-v1-core), so you will need to [grant the Datadog Agent get, list, delete and create access to the ConfigMap resource](/integrations/faq/using-rbac-permission-with-your-kubernetes-integration).

**This functionality is disabled by default**.

To enable leader election you need to:

* Set the variable `leader_candidate` to true in your `kubernetes.yaml` file.

* Use these [Kubernetes RBAC entities](/integrations/faq/using-rbac-permission-with-your-kubernetes-integration) for your Datadog agent to properly configure the previous permissions by [applying this Datadog service account to your pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/):

* Create the ClusterRole, ServiceAccount, and ClusterRoleBinding:
  ```
  kubectl create -f datadog-serviceaccount.yaml
  ```
  Also, you will need to give your GCP user account explicit permission to create Roles and RoleBindings, by giving yourself the cluster-admin role.
  ```
  $ ACCOUNT=$(gcloud info --format='value(config.account)')
  $ kubectl create clusterrolebinding owner-cluster-admin-binding \
      --clusterrole cluster-admin \
      --user $ACCOUNT
  ```
  Without this, creating Roles/ClusterRoles/RoleBindings/ClusterRoleBindings may give you errors.

* Update the dd-agent daemonset config as follows:
  ```
          env:
            - name: KUBERNETES_LEADER_CANDIDATE
              value: "true"
  ...
  ```

5. Then reload the daemonset:
  ```
  kubectl replace --force -f configs/dd-agent/dd-agent.yaml
  ```

Daemonset config errors can be viewed in the Kubernetes log.
