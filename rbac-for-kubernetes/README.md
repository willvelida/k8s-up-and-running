# Table of Contents

- [Role-Based Access Control for Kubernetes](#role-based-access-control-for-kubernetes)
    - [Identity in Kubernetes](#identity-in-kubernetes)
    - [Understanding Roles and RoleBindings](#understanding-roles-and-role-bindings)
    - [Roles and RoleBindings in Kubernetes](#roles-and-role-bindings)
- [Techniques for Managing RBAC](#techniques-for-managing-rbac)
    - [Testing Authorization with can-i](#testing-authorization-with-can-i)
    - [Managing RBAC in Source Control](#managing-rbac-in-source-control)
- [Advanced Topics](#advanced-topics)
    - [Aggregating ClusterRoles](#aggregating-clusterroles)
    - [Using Groups for Binding](#using-groups-for-bindings)

# Role-Based Access Control for Kubernetes

- Role-Based access control provides a mechanism for restricting both access to and actions on Kubernetes APIs to ensure that only authorized users have access.
- RBAC is a critical component to both hard access to the Kubernetes cluster where you are deploying your app, and prevent unexpected accidents where one person the wrong namespace mistakenly takes down production when they think they are destroying their test cluster.
- Every request to Kubernetes is first *authenticated*. Authentication provides the identity of the caller issuing the request.
- Kubernetes does not have a built-in identity store, focusing instead on integrating other identity sources within itself.
- Once users have been authenticated, the authorization phase determines whether they are authorized to perform the request.

## Identity in Kubernetes

- Every request in Kubernetes is associated with some identity.
- Even a request with no identity is associated with the `system:unauthenticated` group.
- Kubernetes makes a distinction between user identities and service account identities.

**Service Accounts** are created and managed by Kubernetes itself and are generally associated with components running inside the cluster.

**User Accounts** are all other accounts associated with actual users of the cluster, and often include automation like continuous delivery services that run outside the cluster.

Kubernetes uses a generic interface for authentication providers. Each of the providers supplies a username and, optionally, a set of groups of which the user belongs. Kubernetes supports a number of authentication providers, including:

- HTTP Basic Auth
- x509 client certificates
- Static token files on host
- Cloud Auth (including Azure AD)
- Authentication webhooks.

You should always use different identities for different applications in your cluster.

You should also have different identities for different clusters.

All of these identities should be machine identities that are not shared with users. You can either use Kubernetes Service Accounts for achieving this, or you can use a Pod identity provider supplied by your identity system.

## Understanding Roles and Role Bindings

Once Kubernetes knows the identity of the request, it needs to determine if the request is authorized for that user. To achieve this, it uses roles and role bindings.

**A role** is a set of abstract capabilities.

**A role binding** is an assignment of a role to one or more identities.

### Roles and Role Bindings

In Kubernetes, two pairs of related resources represent roles and role bindings. Once pair is scoped to a namespace (Role and RoleBinding), while the other pair is scoped to the cluster (ClusterRole and ClusterRoleBinding).

Role resources are namespaced and represent capabilities within that single namespace. *You cannot use namespaced roles for nonnamespaced resources* (e.g. CustomResourceDefinitions), and binding a RoleBinding to a role only provides authorization within the Kubernetes namespace that contains both the Role and RoleBinding.

Here's a simple example of a role that gives an identity the ability to create and modify Pods and Services:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pods-and-services
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["create","delete","get","list","patch","update","watch"]
```

To bind this role to the user `Will`, we need to create a RoleBinding that looks like this. This role binding also binds the group `mydevs` to the same role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pods-and-services
  namespace: default
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: Will
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: mydevs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pods-and-services
```

To do this at the cluster level, we need to create ClusterRole and ClusterRoleBinding resources.

### Using built-in roles

- Designing your own roles can be time-consuming.
- Kubernetes has a large number of built-in cluster roles for well-known system identities that require a known set of capabilities. We can view these by running:

```bash
$ kubectl get clusterroles
```

While most of these built-in roles are for system utilities, four are designed for generic end users:

- The `cluster-admin` role provides complete access to the entire cluster.
- The `admin` role provides complete access to a complete namespace.
- The `edit` role allows an end user to modify resources in a namespace.
- The `view` role allows for read-only access to a namespace.

Most clusters already have numerous ClusterRole bindings set up, and you can view these by running:

```bash
$ kubectl get clusterrolebindings
```

### Auto-reconciliation of built-in roles

When the Kubernetes API starts up, it automatically installs a number of default ClusterRoles that are defined in the code of the API server itself. This means that if you modify any built-in cluster role, those modifications are transient, meaning those roles will be overwritten when the API server restarts.

To prevent this from happening, you need to add the `rbac.authorization.kubernetes.io/autoupdate` annotation with a value of `false` to the built-in ClusterRole resource. This lets the API server know not to overwrite the modified ClusterRole resource.

> [!WARNING]
> By default, the Kubernetes API server installs a cluster role that allows `system:unauthenticated` users access to the API server's API discovery endpoint. For any cluster exposed to a hostile environment (e.g. the public internet) **this is a bad idea!** If you are running a Kubernetes service on the public internet, you should ensure that the `--anonymous-auth=false` flag is set on your API server.

## Techniques for managing RBAC

### Testing Authorization with can-i

The first useful tool is the `auth can-i` command for `kubectl`. This tool is used for testing whether a specific user can perform a specific action.

You can use `can-i` to validate configuration settings as you configure your cluster, or you can ask users to use the tool to validate their access when filing errors or bug reports.

In its simplest usage, the `can-i` command takes a verb and a resource. For example, this command will indicate if the current `kubectl` user is authorized to create Pods:

```bash
$ kubectl auth can-i create pods
yes
```

You can also test subresources like logs or port-forwarding with the ``-subresource` command-line flag:

```bash
$ kubectl auth can-i get pods --subresource=logs
```

### Managing RBAC in Source Control

Like all resources in Kubernetes, RBAC resources are modeled using YAML.

It makes sense to store these resources in version control, which allows for accountability, auditability, and rollback.

The `kubectl` command-line tool provides a `reconcile` command that operates somewhat like `kubectl apply` and will reconcile a set of roles and role bindings with the current state of the cluster. You can run:

```bash
$ kubectl auth reconcile -f some-rbac-config.yaml
```

## Advanced Topics

### Aggregating ClusterRoles

- Sometimes you want to be able to define roles that are combinations of other roles.
- One option would be to simply clone all of the rules from one ClusterRole into another ClusterRole, but this is error-prone and complicated.
- Instead, Kubernetes RBAC supports the usage of an aggregation rule to combine multiple roles in a new role.
- This new role combines all of the capabilities of all of the aggregate roles, and any changes to the consituent subroles will automatically be propogated back into the aggregate role.
- As with other aggregations or groupings in Kubernetes, the ClusterRoles to be aggregated are specified using label selectors.
- In this case, the `aggregationRule` field in the ClusterRole resource contains a `clusterRoleSelector` field, which in turn is a label selector.
- All ClusterRole resources that match this selector are dynamically aggregated into the `rules` array in the aggregate ClusterRole resource.

A best practice for managing ClusterRole resources is to create a number of fine-grained cluster roles and then aggregate them to form higher-level or broader cluster roles.

This is how the built-in cluster roles are defined. For example, you can see that the built-in `edit` role looks like this:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
    name: edit
    ...
aggregationRule:
    clusterRoleSelectors:
    - matchLabels:
        rbac.authorization.k8s.io/aggregate-to-edit: "true"
```

This means that the `edit` role is defined to be the aggregate of all ClusterRole objects that have a label of `rbac.authorization.k8s.io/aggregate-to-edit` set to `true`.

### Using Groups for Bindings

- When managing a large number of people in different organizations with similar access to the cluster, it's generally a best practice to use groups to manage the roles that define access.
- When you bind a group to a Role or ClusterRole, anyone who is a member of that group gains access to the resources and verbs defined by that role.
- To enable any individual to gain access to the group's role, that individual needs to be added to that group.

Using groups is preferred for managing access at scale for several reasons:

1. In any large organization, access to the cluster is defined in terms of the team that someone is part of, rather than their specific identity. Granting privileges to a group makes the association between the specific team and its capabilities clear.
2. Simplicity! When someone joins or leaves a team, it's straightfoward to simply add or remove them to or from a group in a single operation. You also don't have to do lots of work to ensure all team members have the same, consistent set of permissions.

Many groups enable "just in time" access, such that people are only temporarily added to a group in response to an event. This means we can both audit who had access at any particular time and ensure that even a compromised identity can't have access to your production infrastructure.

Finally, in many cases, these same groups are used to manage access to other resources. Using the same groups for access controls to Kubernetes dramatically simplifies management.

To bind a group to a ClusterRole, use a Group kind for the `subject` in the binding:

```yaml
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: my-group-name
```

In Kubernetes, groups are supplied by authentication providers. There is no strong notion of a group within Kubernetes, only that an identity can be part of one or more groups, and those groups can be associated with a Role or ClusterRole via a binding.
