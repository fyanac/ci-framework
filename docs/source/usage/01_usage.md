# Usage guide
The Framework leverages [install_yamls](https://github.com/openstack-k8s-operators/install_yamls)
content and generate the needed bits in order to deploy EDPM on the selected infrastructure.

The Framework will also ensure we're able to reproduce the exact same run we
got in CI with a series of artifacts one may download locally, and re-run.

## Parameters
There are two levels of parameters we may provide:
- top level
- role level

### Top level parameters
The following parameters allow to set a common value for parameters that
are shared among multiple roles:
* `cifmw_basedir`: The base directory for all of the artifacts. Defaults to
`~/ci-framework-data`
* `cifmw_edpm_deploy_baremetal`: (Bool) Toggle whether to deploy edpm on compute nodes
provisioned with bmaas vs pre-provisioned VM.
* `cifmw_installyamls_repos`: install_yamls repository location. Defaults to `../..`
* `cifmw_manifests`: Directory where k8s related manifests will be places. Defaults to
`{{ cifmw_basedir }}/manifests`
* `cifmw_path`: customized PATH. Defaults to `~/.crc/bin:~/.crc/bin/oc:~/bin:${PATH}`
* `cifmw_use_libvirt`: (Bool) toggle libvirt support
* `cifmw_use_crc`: (Bool) toggle rhol/crc usage
* `cifmw_openshift_kubeconfig`: (String) Path to the kubeconfig file if externally provided. If provided will be the kubeconfig to use and update after login.
* `cifmw_openshift_api`: (String) Path to the kubeconfig file. If provided will be the API to authenticate against.
* `cifmw_openshift_user`: (String) Login user. If provided, the user that logins.
* `cifmw_openshift_provided_token`: (String) Initial login token. If provided, that token will be used to authenticate into OpenShift.
* `cifmw_openshift_password`: (String) Login password. If provided is the password used for login in.
* `cifmw_use_opn`: (Bool) toggle openshift provisioner node support.
* `cifmw_use_hive`: (Bool) toggle OpenShift deployment using hive operator.

#### Words of caution
If you want to output the content in another location than `~/ci-framework-data`
(namely set the `cifmw_basedir` to some other location), you will have to update
the `ansible.cfg`, updating the value of `roles_path` so that it includes
this new location.

We cannot do this change runtime unfortunately.

### Role level parameters
Please refer to the README located within the various roles.

## Provided playbooks and scenarios
The provided playbooks and scenarios allow to deploy a full stack with
various options. Please refer to the provided examples and roles if you
need to know more.

## Hooks
The framework is able to leverage hooks located in various locations. Using
proper parameter name, you may run arbitrary playbook or load custom CRDs at
specific points in the standard run.

Allowed parameter names are:
* `pre_infra`: before bootstraping the infrastructure
* `post_infra`: after bootstraping the infrastructure
* `pre_package_build`: before building packages against sources
* `post_package_build`: after building packages against sources
* `pre_container_build`: before building container images
* `post_container_build`: after building container images
* `pre_deploy`: before deploying EDPM
* `post_deploy`: after deploying EDPM
* `post_ctlplane_deploy`: after Control Plane deployment
* `pre_tests`: before running tests
* `post_tests`: after running tests
* `pre_admin_setup`: before admin setup
* `post_admin_setup`: before admin setup
* `pre_reporting`: before running reporting
* `post_reporting`: after running reporting

Since steps may be skipped, we must ensure proper post/pre exists for specific
steps.

In order to provide a hook, please pass the following as an environment file:
```YAML
pre_infra:
    - name: My glorious hook name
      type: playbook
      source: foo.yml
    - name: My glorious CRD
      type: crd
      host: https://my.openshift.cluster
      username: foo
      password: bar
      wait_condition:
        type: pod
      source: /path/to/my/glorious.crd
```
In the above example, the `foo.yml` is located in
[ci_framework/hooks/playbooks](https://github.com/openstack-k8s-operators/ci-framework/tree/main/ci_framework/hooks/playbooks) while
`glorious.crd` is located in some external path.

Also, the list order is important: the hook will first load the playbook,
then the CRD.

Note that you really should avoid pointing to external resources, in order to
ensure everything is available for job reproducer.

## Ansible tags
In order to allow user to run only a subset of tasks while still consuming the
entry playbook, the Framework exposes tags one may leverage with either `--tags`
or `--skip-tags`:

* `bootstrap`: Run all of the package installation tasks as well as the potential system configuration depending on the options you set.
* `packages`: Run all package installation tasks associated to the options you set.

For instance, if you want to bootstrap a hypervisor, and reuse it over and
over, you'll run the following commands:
```Bash
$ ansible-playbook deploy-edpm.yml -K --tags bootstrap,packages [-e @scenarios/centos-9/some-environment -e <...>]
$ ansible-playbook deploy-edpm.yml -K --skip-tags bootstrap,packages [-e @scenarios/centos-9/some-environment -e <...>]
```

Running the commande twice, with `--tags` and `--skip-tags` as only difference,
will ensure your environment has the needed directories, packages and
configurations with the first run, while skip all of those tasks in the
following runs. That way, you will save time and resources.

More tags may show up according to the needs.