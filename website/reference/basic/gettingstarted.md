---
layout: mcollective
title: Getting Started
disqus: true
---
[Screencasts]: /introduction/screencasts.html
[ActiveMQ]: http://activemq.apache.org/
[EC2Demo]: /introduction/ec2demo.html
[Stomp]: http://stomp.codehaus.org/Ruby+Client
[DepRPMs]: http://www.marionette-collective.org/activemq/
[Gentoo]: /reference/os/gentoo.html
[DebianBug]: http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=562954
[SecurityWithActiveMQ]: /reference/integration/activemq_security.html
[ActiveMQClustering]: http://www.devco.net/archives/2009/11/10/activemq_clustering.php
[ActiveMQSamples]: http://github.com/mcollective/marionette-collective/tree/master/ext/activemq/examples/
[ConfigurationReference]: /reference/basic/configuration.html
[Terminology]: /introduction/terminology.html
[SimpleRPCIntroduction]: /simplerpc/
[ControllingTheDaemon]: /reference/basic/daemon.html
[SSLSecurityPlugin]: /reference/plugins/security_ssl.html
[ConnectorStomp]: /reference/plugins/connector_stomp.html
[MessageFlowCast]: /introduction/screencasts.html#message_flow
[Plugins]: http://code.google.com/p/mcollective-plugins/

# {{page.title}}

 * TOC Placeholder
 {:toc}

Below find a rough guide to get you going, this assumes the client and server is on the same node, but servers don't need the client code installed.

For an even quicker intro to how it all works you can try our [EC2 based demo][EC2Demo]

Look at the [Screencasts] page, there are [some screencasts dealing with basic architecture, terminology and so forth][MessageFlowCast] that you might find helpful before getting started.

## Requirements
We try to keep the requirements on external Gems to a minimum, you only need:

 * A Stomp server, tested against [ActiveMQ]
 * Ruby
 * Rubygems
 * [Ruby Stomp Client][Stomp]

RPMs for these are available [here][DepRPMs].

Information on installing mcollective and stomp from within Gentoo's portage are available [here][Gentoo].

**NOTE: You need version Stomp Gem 1.1 for mcollective up to 0.4.5.  mcollective 0.4.6 onward supports 1.1 and 1.1.6 and newer**


## ActiveMQ
I've developed this against ActiveMQ.  It should work against other Stomp servers but I suspect if you choose 
one without username and password support you might have problems, please let me know if that's the case 
and I'll refactor the code around that.

Full details on setting up and configuring ActiveMQ is out of scope for this, but you can follow these simple 
setup instructions for initial testing (make sure JDK is installed, see below for Debian specific issue regarding JDK):

### Download and Install
 1. Download the ActiveMQ "binary" package (for Unix) from [ActiveMQ]
 1. Extract the contents of the archive:
 1. cd into the activemq directory
 1. Execute the activemq binary 

{% highlight console %}
   % tar xvzf activemq-x.x.x.tar.gz
   % cd activemq-x.x.x
   % bin/activemq
{% endhighlight %}

Below should help you get stomp and a user going. For their excellent full docs please see [ActiveMQ].  
There are also tested configurations in [the ext directory][ActiveMQSamples]

A spec file can be found in the *ext* directory on GitHub that can be used to build RPMs for RedHat/CentOS/Fedora 
you need *tanukiwrapper* which you can find from *jpackage*, it runs fine under OpenJDK that comes with recent 
versions of these Operating Systems.  I've uploaded some RPMs and SRPMs [here][DepRPMs].

For Debian systems you'd be better off using OpenJDK than Sun JDK, there's a known issue [#562954][DebianBug].

### Configuring Stomp
First you should configure ActiveMQ to listen on the Stomp protocol

And then you should add a user or two, to keep it simple we'll just add one user, the template file will hopefully make it obvious where this goes, it should be in the _broker_ block:

*Note: This config is for ActiveMQ 5.3*

{% highlight xml %}
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:amq="http://activemq.apache.org/schema/core"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd   
  http://activemq.apache.org/camel/schema/spring http://activemq.apache.org/camel/schema/spring/camel-spring.xsd">

   <broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" useJmx="true">
    <plugins>
      <simpleAuthenticationPlugin>
        <users>
          <authenticationUser username="mcollective" password="marionette" groups="systemusers,everyone"/>
        </users>
      </simpleAuthenticationPlugin>

      <authorizationPlugin>
        <map>
          <authorizationMap>
            <authorizationEntries>
              <authorizationEntry topic="mcollective.>" write="systemusers" read="systemusers" admin="systemusers" />
              <authorizationEntry topic="ActiveMQ.Advisory.>" read="everyone,all" write="everyone,all" admin="everyone,all"/>
            </authorizationEntries>
          </authorizationMap>
        </map>
      </authorizationPlugin>
    </plugins>

    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://0.0.0.0:6166"/>
       <transportConnector name="stomp"   uri="stomp://0.0.0.0:6163"/>
    </transportConnectors>
   </broker>
</beans>
{% endhighlight %}

This creates a user *mcollective* with the password *marionette* and give it access to read/write/admin */topic/mcollective.`*`*.

Save the above code as activemq.xml and run activemq as - if installing from a package probably _/etc/activemq/activemq.xml_:

If you did not install from RPM or deb then start by hand:

{% highlight console %}
  $ activemq xbean:/path/to/mcollective.xml
{% endhighlight %}

Else your package would have a RC script:

{% highlight console %}
  # /etc/init.d/activemq start
{% endhighlight %}

For further info about ActiveMQ settings you might need see [SecurityWithActiveMQ] and [ActiveMQClustering].

There are also a few known to work and tested [configs in git][ActiveMQSamples].

## mcollective
### Download and Extract
Grab a copy of the mcollective ideally you'd use a package for your distribution else there's a tarfile that 
you can use, you can extract it wherever you want, the RPMs or deps will put files in Operating System compatible 
locations.  If you use the tarball you'll need to double check all the paths in the config files below.

### Configure
You'll need to tweak some configs in */etc/mcollective/client.cfg*, a full reference of config settings can be 
found [here][ConfigurationReference]:

Mostly what you'll need to change is the *identity*, *plugin.stomp.`*`* and the *plugin.psk*:

{% highlight ini %}
  # main config
  topicprefix = /topic/mcollective
  libdir = /usr/libexec/mcollective
  logfile = /dev/null
  loglevel = debug
  identity = fqdn
  
  # connector plugin config
  connector = stomp
  plugin.stomp.host = stomp.your.net
  plugin.stomp.port = 6163
  plugin.stomp.user = unset
  plugin.stomp.password = unset
  
  # security plugin config
  securityprovider = psk
  plugin.psk = abcdefghj
{% endhighlight %}

You should also create _/etc/mcollective/server.cfg_ here's a sample, , a full reference of config settings can be found here [ConfigurationReference]:

{% highlight ini %}
  # main config
  topicprefix = /topic/mcollective
  libdir = /usr/libexec/mcollective
  logfile = /var/log/mcollective.log
  daemonize = 1
  keeplogs = 1
  max_log_size = 10240
  loglevel = debug
  identity = fqdn
  registerinterval = 300
  
  # connector plugin config
  connector = stomp
  plugin.stomp.host = stomp.your.net
  plugin.stomp.port = 6163
  plugin.stomp.user = mcollective
  plugin.stomp.password = password
  
  # facts
  factsource = yaml
  plugin.yaml = /etc/mcollective/facts.yaml
  
  # security plugin config
  securityprovider = psk
  plugin.psk = abcdefghj
{% endhighlight %}

Replace the *plugin.stomp.host* with your server running ActiveMQ and replace the *plugin.psk* with a Pre-Shared Key of your own.

The STOMP connector supports other options like failover pools, see [ConnectorStomp] for full details.

### Create Facts
By default - and for this setup - we'll use a simple YAML file for a fact source, later on you can use Reductive Labs Facter or something else.

Create */etc/mcollective/facts.yaml* along these lines:

{% highlight yaml %}
  ---
  location: devel
  country: uk
{% endhighlight %}

### Start the Server
If you installed from a package start it with the RC script, else look in the source you'll find a LSB compatible RC script to start it.

### Test from a client
If all is setup you can test with the client code:

{% highlight console %}
% mc-ping 
your.domain.com                           time=74.41 ms

---- ping statistics ----
1 replies max: 74.41 min: 74.41 avg: 74.41
{% endhighlight %}

This sent a simple 'hello' packet out to the network and if you started up several of the *mcollectived.rb* processes on several machines you 
would have seen several replies, be sure to give each a unique *identity* in the config.

At this point you can start exploring the discovery features for example:

{% highlight console %}
% mc-find-hosts --with-fact country=uk
your.domain.com
{% endhighlight %}

This searches all systems currently active for ones with a fact *country=uk*, it got the data from the yaml file you made earlier.

If you use confiuration management tools like puppet and the nodes are setup with classes with *classes.txt* in */var/lib/puppet* then you 
can search for nodes with a specific class on them - the locations will configurable soon:

{% highlight console %}
% mc-find-hosts --with-class common::linux
your.domain.com
{% endhighlight %}

Chef does not yet support such a list natively but we have some details on the wiki to achieve the same with Chef.

The filter commands are important they will be the main tool you use to target only parts of your infrastructure with calls to agents.

See the *--help* option to the various *mc-`*`* script for available options.  You can now look at some of the available plugins and 
play around, you might need to run the server process as root if you want to play with services etc.

### Plugins
We provide limited default plugins, you can look on our sister project [MCollective Plugins][Plugins] where you will 
find various plugins to manage packages, services etc.

### Further Reading
From here you should look at the rest of the wiki pages some key pages are:

 * [Screencasts] - Get a hands-on look at what is possible
 * [Terminology]
 * [Introduction to Simple RPC][SimpleRPCIntroduction] - a simple to use framework for writing clients and agents
 * [ControllingTheDaemon] - Controlling a running daemon
 * [SSLSecurityPlugin] - Using SSL for secure message signing and authentication of clients
 * [ConnectorStomp] - Full details on the Stomp adapter including failover pools
