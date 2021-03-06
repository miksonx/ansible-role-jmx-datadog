# Ansible Role JMX Datadog

![Build Status](https://travis-ci.org/peopledoc/ansible-role-jmx-datadog.svg?branch=master)

Overview
--------

Ansible role to setup JMX credentials and to add Datadog java agent.

**You need to add `Datadog.datadog` to the requirements.yml of your project**. Cf [requirements.yml](requirements.yml).

Usage
-----

Add the role to your playbook :

```yaml
roles:
  - role: ansible-role-jmx-datadog
    vars:
      jmx_instances:
        - port: 7199
          user: username
          password: changeit
          name: app1 # service name on APM
        - port: 7299
          user: username2
          password: changeit2
          name: app2 # service name on APM
      jvm_user: user
      jvm_group: group
      jvm_user_id: 1000 # optional
      jvm_group_id: 1000 # optional

      datadog_api_key: "{{ vault_datadog_api_key }}"
      datadog_version: "6.1.4"
      datadog_java_integration: yes
      datadog_config:
        hostname: host.domain
        tags: "env:{{ inventory_dir | basename }}"
        ...
```

If the ansible remote user is not `root` this role might fail. You can add
`become: true` to the role invocation in that case.

This role has the [Datadog Ansible Role](https://github.com/DataDog/ansible-datadog) as
a dependency and will configure the jmx check for it.
You can add Datadog role parameters as parameters of this role (like `datadog_api_key`).

You can also skip the Datadog role with `--skip-tags datadog_agent` to just setup JMX.

Be aware that the agent v6 can't work without systemd. So it can't be installed on Debian Wheezy.
(you can still install the v5 for now: [agent v5](https://github.com/DataDog/ansible-datadog#agent-5-older-version))

Parameters
----------

### Mandatory

* `jmx_instances` with :
  * `user` and `password` are the username/password pair for JMX (with readonly access).
  They are mandatory. The same ones will be used with the Datadog integration. You can use
  them for other JMX usages if needed.
  * `name` is used to define service name for Datadog in the JMX integration
    and with the Datadog java agent (`-Ddd.service.name`).
  * `port` is the port used by JMX for this instance.

* `jvm_user` and `jvm_group` are the user/group who will run the application using JMX
(change access rights on default `jmxremote.password` and `jmxremote.access` files in `$JAVA_HOME/jre/lib/management`).

* `datadog_version` is the version of the agent. You can also define
  `datadog_agent_version` from the [Datadog Ansible Role](https://github.com/DataDog/ansible-datadog/blob/master/README.md#role-variables) but `datadog_version` takes precedence.

* You have to add some parameters for the Datadog role like `datadog_api_key`. Please read the [Datadog Ansible Role documentation](https://github.com/DataDog/ansible-datadog/).

### Optional

* `jvm_user_id` and `jvm_group_id` are used to configure UID and GID of `jvm_user` if the user doesn't already exist.
  You can safely skip using these parameters if `jvm_user` already exists on the host.

* `datadog_java_integration: no` can be used to remove Datadog Java agent (the JMX integration will still
  be configured). You'll have to use another agent with the JMX parameters provided to this role.

* `datadog_java_agent_version` is the version used to fetch the [Datadog Java agent](https://github.com/datadog/dd-trace-java) from [Central](https://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22com.datadoghq%22%20AND%20a%3A%22dd-java-agent%22). Defaults to `LATEST`.

If you want to add more datadog checks configurations, you'll have to use the
`other_datadog_checks` variable as `datadog_checks` can't be override because
it is already in use as a role dependency parameter. The role will combine
`other_datadog_checks` and the check for jmx to obtain `datadog_checks`.

```yaml
roles:
  - role: ansible-role-jmx-datadog
    vars:
    ...
    other_datadog_checks:
      nginx:
        instances:
          ...
```

JVM parameters
--------------

The JVM parameters needed to use JMX are registered in the `java_opts_datadog_jmx` ansible
variable for later reuse in a playbook:

```
-Dcom.sun.management.jmxremote.port=7199 -Dcom.sun.management.jmxremote.ssl=false
-javaagent:/usr/local/bin/dd-java-agent.jar -Ddd.service.name=instance-name-on-datadog-interface
```

These parameters are also accessible as a list if needed in `java_opts_datadog_jmx_list`. You can use the following to add the JVM options:

```yaml
- name: task or role running jvm
  environment:
    - JAVA_TOOL_OPTIONS: "{{ java_opts_datadog_jmx }}"
```

SSL configuration
-----------------

This role doesn't setup SSL for JMX. You can safely disable it if the datadog
agent and the application using JMX are on the same host (the scenario assumed
by this role). If you setup the datadog agent on another host, you should setup SSL
for JMX using
[this documentation](https://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html#SSL_enabled)
and add configuration from [this documentation](https://docs.datadoghq.com/integrations/java/).

Tests
-----

Tests can be executed using:

`$ molecule --base-config molecule/base.yml test --driver-name docker --all`

The dependencies are ansible, molecule and docker Python packages.

License
-------
BSD
