---
layout: post
title: "Creating Custom Ambari Services"
date: 2015-09-11 13:25:09 -0500
comments: true
categories: ambari
---

[Apache Ambari](https://ambari.apache.org/) is a cluster management UI that
simplifies the process of installing, configuring, and monitoring services
from the [Hortonworks Data Platform](http://hortonworks.com/hdp/whats-new/),
e.g. Hadoop, Hive, Kafka, etc.  While HDP contains many of the services you
might want on a cluster, you might find that you need something that HDP
doesn't offered, such as a notebook server like Jupyter, but that you want to
manage via Ambari.  For this purpose, Ambari allows you to create custom
services, which Ambari then treats exactly like any of the built-in services.

The main problem with creating a custom service is that
[the documentation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=38571133)
has rather large gaps in information that make it difficult to get started.
For example, the section on adding configuration to a service gives you a nice
XML file with an example property, but fails to explain how to translate that
property into changes into the various configuration files on which your
services rely.

In this post, I attempt to fill in the holes in the documentation and to
explain the various "gotcha's" I encountered while trying to create an Ambari
service for [Hue](http://gethue.com).  Note that I've been using Ambari 2.1.1,
but this should apply to Ambari 1.5.0 or higher.

<!-- more -->

Service Files
-------------

The basic idea behind a creating a custom service is that you'll install on
the machine running the Ambari server a number of files with a particular
directory structure.  The Ambari documentation lists most of the relevant
files, but omits some important ones, as well as mentioning some that aren't
all that useful for most people.  The basic structure of a simple service
should look like this:

```
|_ <service name>
   metainfo.xml
   |_ configuration
      <configuration XML files>
   |_ package
      |_ scripts
         <component>.py
         params.py
      |_ templates
```

This service directory should be placed on the Ambari server host in the
`services` directory of the appropriate stack directory, which are all located
in `/var/lib/ambari-server/resources/stacks/<stack name>/<stack version>`.
For example, for use in the HDP 2.3 stack, place the custom service directory
in `/var/lib/ambari-server/resources/stacks/HDP/2.3/services`.  When the files
are in place, you can install the service in the Ambari UI after restarting
the server, e.g. `service ambari-server restart`.


Metadata
--------

The first step in defining your custom service is to create the service
metadata in the file `metadata.xml`.  The main things you have to define are:

- Service name and version, as well as nice things like a display name and a
  description
- The various components that'll be running on your cluster
- Any configuration files associated with those components that will be
  managed by Ambari
- Any packages you need to install, usually the package that contains the
  service you're going to run
- Dependencies, both on other services and other Ambari configs that, when
  changed, will necessitate a restart of your service

Here's a simple metadata file for a service named `MY_SERVICE` with a server,
`SERVICE_SERVER`, and a client, `SERVICE_CLIENT`:

```xml
<?xml version="1.0"?>
<metainfo>
    <schemaVersion>2.0</schemaVersion>
    <services>
        <service>
            <name>MY_SERVICE</name>
            <displayName>My Custom Service</displayName>
            <comment>A custom service that I created</comment>
            <version>1.2.3</version>

            <components>
                <component>
                    <name>SERVICE_SERVER</name>
                    <displayName>A server</displayName>
                    <category>MASTER</category>
                    <cardinality>1</cardinality>
                    <dependencies>
                        <name>HDFS/HDFS_CLIENT</name>
                        <scope>host</scope>
                        <auto-deploy>
                            <enabled>true</enabled>
                        </auto-deploy>
                    </dependencies>
                    <commandScript>
                        <script>scripts/server.py</script>
                        <scriptType>PYTHON</scriptType>
                        <timeout>600</timeout>
                    </commandScript>
                    <configFiles>
                        <configFile>
                            <type>properties</type>
                            <fileName>service.conf</fileName>
                            <dictionaryName>service-conf</dictionaryName>
                        </configFile>
                    </configFiles>
                </component>

                <component>
                    <name>SERVICE_CLIENT</name>
                    <displayName>A client</displayName>
                    <category>CLIENT</category>
                    <cardinality>1+</cardinality>
                    <commandScript>
                        <script>scripts/client.py</script>
                        <scriptType>PYTHON</scriptType>
                    </commandScript>
                </component>
            </components>

            <osSpecifics>
                <osSpecific>
                    <osFamily>any</osFamily>
                    <packages>
                        <package>
                            <name>my_service</name>
                        </package>
                    </packages>
                </osSpecific>
            </osSpecifics>

            <requiredServices>
                <service>HDFS</service>
            </requiredServices>

            <configuration-dependencies>
                <config-type>core-site</config-type>
            </configuration-dependencies>
        </service>
    </services>
</metainfo>
```

I've added a dependency on the `HDFS` service, and also required that the HDFS
client be installed on the same machine as the server.  Here's a brief
explanation of some of the part of the metadata file:

- A component can either be a `MASTER`, a `SLAVE`, or a `CLIENT`, which mostly
  determines what commands are available to Ambari to manage that component.
  Generally, servers are `MASTER` components and are only installed once,
  `SLAVE` components are installed on multiple hosts and can be horizontally
  scaled, and `CLIENT` components are command-line clients that should be
  installed on hosts in which a user is likely to login
- Cardinality is the number of hosts onto which the component will be
  installed; this will usually be `1` for a `MASTER` and `1+` for a `CLIENT`
  or `SLAVE`
- Each `configFile` must have a corresponding file in the `configuration`
  directory with the name `<dictionaryName>.xml`.  The file must always be
  XML, regardless of the type of config that will actually be installed on the
  host.  For example, a config file of type `env` will have an XML file in the
  `configuration` directory
- The command script is a script, typically written in Python, that Ambari
  will use to manage your service, e.g. starting/stopping the process,
  checking status, updating configuration
- Configuration dependencies always refer to the dictionary name


Scripts
-------

Next, define the component scripts in `package/scripts`.  Here's an example
`server.py` script:

```python
from resource_management.libraries.script.script import Script
from resource_management.core.resources.system import File, Execute
from resource_management.core.source import Template

import os.path

CONF_DIR = '/etc/service'

class Server(Script):

    def install(self, env):
        self.install_packages(env)

    def configure(self, env):
        import params
        env.set_params(params)

        File(os.path.join(CONF_DIR, 'service.conf'),
             content=Template('service.conf.j2'),
             mode=0644,
             owner='root')

    def stop(self, env, rolling_restart=False):
        Execute(('service', 'service-server', 'stop'))

    def start(self, env, rolling_restart=False):
        self.configure(env)
        Execute(('service', 'service-server', 'start'))

    def status(self, env):
        Execute(('service', 'service-server', 'status'))


if __name__ == '__main__':
    Server().execute()
```

This script defines all of the methods that a `MASTER` component needs:
`install`, `configure`, `stop`, `start`, and `status`.  The `client.py` script
will be similar, except that `CLIENT` components don't need the `start` and
`stop` methods.  Note that the `configure` method imports a module `params`,
defined in `package/scripts/params.py`, and generates a config file from a
template, `service.conf.j2`;  I'll discuss these next.


Configuration
-------------

Configuration files define parameters for your service that can be tweaked in
Ambari.  Typically, this will be a 1-to-1 mapping to the settings that can be
altered in the corresponding service config file.  In the metadata file, you
can see that the service has a single configuration file, `service.conf`.
Let's say that it's a simple properties file that looks something like this:

```
# Required
host = 0.0.0.0
port = 12345

# Optional
# user =
```

We'll need to mirror those properties in `configuration/service-conf.xml`:

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration supports_final="true">

  <property>
      <name>host</name>
      <description>The host running the server</description>
      <value>0.0.0.0</value>
  </property>

  <property>
      <name>port</name>
      <description>The port on which the server is running</description>
      <value>12345</value>
  </property>

  <property>
      <name>user</name>
      <description>The user as whom to run the server</description>
  </property>

</configuration>
```

Each property needs a name and a description.  You can also pass in a default
value if you like.  However, things get tricky if you want to either require
user input or want optional properties.  If you add an empty value, the Ambari
UI will refuse to proceed through installation until you specify a value;
however, if you omit the `value` tag entirely, Ambari will consider it
optional.  Note that if you type something the input box for an optional value
and then delete it, the Ambari UI will complain.

Now, make a Jinja 2 template file in `package/templates`; you can call it
anything you want, but I recommend giving it the same filename that you
specified in the metadata, except with an additional `.j2` suffix.  This
template will be used to generate the config file on any hosts that require
it.  If you need a refresher on how to write Jinja 2 templates,
[here's a link to the documentation](http://jinja.pocoo.org/docs/dev/templates/).

Here how the template file `package/templates/service.conf.j2` should look:

{% raw %}
```
host = {{ host }}
port = {{ port }}

{% if user %}
user = {{ user }}
{% endif %}
```
{% endraw %}

You might think that the variables in the curly brackets are replaced using
the values from the configuration XML file, but that's not quite true.  In
fact, you have to bridge that gap using another python file:
`package/scripts/params.py`.  This script's purpose is to populate variables
to be used in templates and other scripts.  Typically, you will either
generate values directly or you will load them from Ambari's current
configuration, which is initially populated from the configuration XML file.

Let's take a look at a simple `params.py` example:

```python
from resource_management.libraries.script.script import Script

# Ambari configuration
script_config = Script.get_config()

# Service configurations, e.g. core-site.xml
configs = script_config['configurations']

# Host information, e.g. what services are installed on which hosts
host_info = script_config['clusterHostInfo']

# Access configs using the dictionary name from the metainfo.xml file
service_conf = configs['service-conf']

# Set template variables; keys are the configuration XML property names
host = service_conf['host']
port = service_conf['port']
user = service_conf.get('user')
```

This parameter file is pretty straightforward.  The only thing that can be
confusing is the `clusterHostInfo`; the best way to learn what's in here is to
either look at prior art in the
[Ambari code](https://github.com/apache/ambari/tree/trunk/ambari-server/src/main/resources/common-services),
or to inspect the dictionary contents directly.  The only way I know of to do
the latter is to print it in `params.py`, restart the Ambari server, and then
install the service.  When Ambari starts the server, can click on the 'service
start' operation to see the output of the script, which will contain the
contents of the `clusterHostInfo` dictionary if you printed it.


Troubleshooting
---------------

If you've created all of your service files properly, you should be able to
restart the Ambari server and then install your service using the Ambari UI.
However, chances are that you made a mistake somewhere.  Here are some common
issues I've encountered in the past:

- If the 'service start' step fails with warnings, check the output of that
  operation in the Ambari UI;  you should see a fairly obvious stack trace.
  You might have failed to define some variables in `params.py` that are
  needed in your template, there could be a syntax error in one of your
  scripts or templates, or there could be another problem entirely.
- If the installation completes, but your service fails to start, check that
  any configuration files that you created look correct.  Check any logs that
  your service writes, or start the service manually.  Chances are that it's
  mis-configured

If you made an error and need to make changes to any of your files, you'll
likely need to re-install the service.  However, this is often a big pain, as
there's no 'uninstall service' button in the Ambari UI.  You'll have to use
the Ambari API, which is easier if you use the
[Python Ambari client](https://github.com/jimbobhickville/python-ambariclient).  

In theory, you can uninstall your service by first stopping all components in
the UI, then deleting the service using the python client:

```python
from ambariclient.client import Ambari

client = Ambari(ambari_host,
                port=80,
                username='my-user',
                password='my-password')

client.clusters('your_cluster_id').services('MY_SERVICE').delete()
```

However, this will often fail due to a bug in Ambari that won't allow it to
recognize a service as stopped if it never started successfully.  This can
happen if any warnings appeared during the service install, e.g. if you had
syntax errors in one of your scripts.  If you see any warnings in the service
install, then you're pretty much out of luck; you'll have to delete the
cluster and rebuilt it (hopefully, you were using a blueprint!).

If you don't see any warnings in the installation, but the service still
appears as "stopped" in the Ambari UI, you might have simply written one of
your configuration files incorrectly; this will usually appear in your service
logs or if you try to start the service manually.  You'll still have the same
issue deleting the service, but you can get around it by starting the service
again in the Ambari UI, then quickly running the `delete` command in the
Python client before Ambari checks the service status.


Final Notes
-----------

That's all you need to know to create a simple custom Ambari service.
However, there's much more you can do, e.g. creating alerts, including static
files, making your service cross-platform, etc.  I recommend you look at
[existing Ambari services](https://github.com/apache/ambari/tree/trunk/ambari-server/src/main/resources/common-services)
for inspiration.
