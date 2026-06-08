# Cluster templates

## What is a cluster template?

A cluster template is an [Ansible role] that is responsible for installing and configuring an OpenShift cluster. Cluster templates are distributed in Ansible [collections]. A collection may contain multiple templates.

[collections]: https://docs.ansible.com/ansible/latest/dev_guide/developing_collections.html
[ansible role]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html

## How to create a cluster template

### Create a collection

If you don't already have a collection for distributing your cluster templates, you can create a new one using the `ansible-galaxy collection init` command. Collection names consist of two parts, a namespace (often a company name) and a name, so if we wanted to create a new template named `redhat.rhoai`, we would run:

```
ansible-galaxy collection init redhat.rhoai
```

That creates a directory tree that looks like:

```
redhat/
└── rhoai
    ├── docs
    ├── galaxy.yml
    ├── meta
    │   └── runtime.yml
    ├── plugins
    │   └── README.md
    ├── README.md
    └── roles
```

### Create a role

A cluster template role consists of at least two playbooks:

- `tasks/main.yaml` -- this playbook runs with privileges on the management cluster. It is responsible for creating the cluster when `cluster_template_state` is `present` and for deleting the cluster when `cluster_template_state` is `absent`.

- `tasks/post-install.yaml` -- this playbook runs with privileges on the tenant cluster. It is responsible for performing any software installation and configuration tasks on the tenant cluster.

A template role also contains two metadata files:

- `meta/argument_specs.yaml` -- this file describes all the parameters accepted by the role.

- `meta/cloudkit.yaml` -- this file contains metadata such as a friendly title and a description.

### Writing argument_specs.yaml

The syntax of the `argument_spec.yaml` file is fully documented in the [Ansible documentation][argspec]. Your role will need to accept a `cluster_order` parameter with the values from the cluster request, and a `template_parameters` parameter with any values required by your role (or by roles to which you call out). A basic example looks like this:

```yaml
argument_specs:
  main:
    options:
      cluster_order:
        type: dict
        required: true
      cluster_working_namespace:
        type: str
        required: true
      node_requests:
        type: list
        elements: dict
        options:
          numberOfNodes:
            type: int
            required: true
          resourceClass:
            type: str
            required: true
            choices:
            - fc430
      template_parameters:
        type: dict
        options:
          pull_secret:
            description: >
              The pull secret contains credentials for authenticating to image repositories.
            type: str
            required: true
          ssh_public_key:
            description: >
              A public ssh key that will be installed into the `authorized_keys` file
              of the `core` user on cluster worker nodes.
            type: str
```

[argspec]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-argument-validation

### Writing cloudkit.yaml

The `cloudkit.yaml` file contains metadata about your template role that isn't part of the standard Ansible metadata. Here is a sample `cloudkit.yaml`:

```yaml
# Define the display name and description of the template
title: Simple OpenShift 4.17 Cluster
description: >
  This template will build a minimally configured OpenShift 4.17 cluster.

# This sets the nodeRequest field in the ClusterOrder manifest. There must be at
# least one entry in this list.
default_node_request:
- resourceClass: fc430
  numberOfNodes: 2

# This declares that for a cluster built from this template, only the resource
# classes listed here may be used when requesting additional nodes. If this
# list is empty or unspecified, a ClusterOrder may request nodes from any
# available resource class.
allowed_resource_classes:
- fc430
```

### Example main.yaml

In most cases, your `main.yaml` will simply include roles distributed as part of the OSAC installer. A typical example would be:

```
- name: Create a new instance of this template
  when: cluster_template_state == 'present'
  ansible.builtin.import_role:
    name: cloudkit.templates.ocp_4_17_small

- name: Destroy an instance of this template
  when: cluster_template_state == 'absent'
  ansible.builtin.import_role:
    name: cloudkit.templates.ocp_4_17_small
```

### Example post-install.yaml

The `post-install.yaml` task list runs with credentials for the tenant cluster and will typically consist of many calls to the [`kubernetes.core.k8s`](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html) module:

```
- name: Create grafana namespace
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: grafana
```

You can leverage [Kustomize] to deploy resources by using the [`kubernetes.core.kustomize`](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/kustomize_lookup.html) lookup:

```
- name: Install Operators
  kubernetes.core.k8s:
    state: present
    definition: "{{ lookup('kubernetes.core.kustomize', dir = kustomize_dir ~ '/operators') }}"
    validate_certs: false
```

## Using a cluster template

There are two steps to making your collection available to the OSAC installer:

1. Build a new [execution environment] that includes your collection.
2. Configure the `cloudkit_template_collections` variable to include the name of your collection

[execution environment]: https://docs.ansible.com/ansible/latest/getting_started_ee/index.html

### Build a new execution environment

1. Check out the `cloudkit-aap` repository:

    ```
    git clone https://github.com/innabox/cloudkit-aap && cd cloudkit-aap
    ```

2. Edit `collections/requirements.yml` to include your new collection. If your collection has been published to Ansible Galaxy, you can use the fully qualified collection name:

    ```
    - name: redhat.rhoai
      version: 1.0.0
    ```

    If your collection is hosted in a Git repository, provide the URL to the repository:

    ```
    - source: https://github.com/innabox/redhat-rhoai-collection.git
      type: git
      version: 1.0.0
    ```

    If you are not tagging versions in your repository, you can use a commit hash instead of a tag.

3. Build a new execution environment:

    ```
    cd execution-environment
    ansible-builder build --tag cloudkit-aap-ee
    ```

4. Publish the container image to a container registry:

    ```
    podman push cloudkit-aap-ee quay.io/myusername/cloudkit-aap-ee:latest
    ```

### Configure the installer to use the new image


Configure AAP to use the new image as the execution environment and to look for templates in your Ansible collection.. You will need to set the `AAP_EE_IMAGE` key in the `config-as-code-ig` Secret to the fully qualified image name. For example, if you are deploying using the installer repository, you could add the following to your `kustomization.yaml`:

    ```
    secretGenerator:
      - name: config-as-code-ig
        options:
          disableNameSuffixHash: true
        literals:
          - AAP_EE_IMAGE=quay.io/myname/cloudkit-aap-ee@sha256:ddc3dec0315de244c14df9d2ee11cc242a60dabd0b0b43b2c48fd1a0de2e61da
          - CLOUDKIT_TEMPLATE_COLLECTIONS=cloudkit.templates,redhat.rhoai
    ```

    Now, re-deploy the installation manifests.

