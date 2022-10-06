# The Nasty Project

This is an example of a "nasty" project that does all wrong.

It is a very basic project, that uses groovy-eclipse-compiler to compile
sources (set up as explained here https://github.com/groovy/groovy-eclipse/wiki/Groovy-Eclipse-Maven-plugin).

Moreover for example sake, it depends on Atlassian Audit API artifact `com.atlassian.audit:atlassian-audit-api` 
not available from Maven Central, but from Atlasian Maven Proxy (also set up as explained here 
https://developer.atlassian.com/server/framework/atlassian-sdk/atlassian-maven-repositories-2818705/).

Finally, to make this example "even worse", this project uses custom `settings.xml`.

To build this "nasty" project, do this (use pristine local repository, to force Maven download
all the needed artifacts):

```
$ mvn -s setting.xml clean package -Dmaven.repo.local=local
```

The build time (only for relative purposes, depends a lot of workstation, internet speed, etc) for me
took 02:05 min as reported by Maven. But, just take a peek at full Maven console
output... (sit down before).

# What is happening?

You will notice that all your dependencies are downloaded from Atlassian repository (but same
would happen with Groovy repository, if that would be the first). Reason is pretty much straight-
forward: **both these repositories are "group" repositories**, that contain Maven Central among
their members as well, hence, all artifacts you expect from MC can be obtained from these
repositories as well! As Maven goes "round robin" on ordered list of remote repositories, it will
find everything in first, Atlassian repository. Hence, as an side-effect, you are basically getting 
MC Artifacts via Atlassian infrastructure, or Groovy repository (as same stands for that as well). 
Not only is slower, but not to mention the danger side of all this.

# Lets fix this

This is where Remote Repository Filtering comes to play. We can fix this ONLY by filtering
(while this "nasty" project does several things wrongly, we will NOT touch any of the POM or
custom settings XML to achieve this fix).

For this, you need following things:

* local build of maven-resolver PR https://github.com/apache/maven-resolver/pull/197
* local build of maven-3.9.x branch (w/ change to use local maven-resolver build)
* deployed the custom Maven build somewhere

After all that above, let's build this nasty project using following commands (assuming
you unpacked custom Maven build to ~/tmp/apache-maven-3.9.0-SNAPSHOT directory):

```
$ ~/tmp/apache-maven-3.9.0-SNAPSHOT/bin/mvn -s settings.xml -Dmaven.repo.local=flocal clean package \
  -Daether.remoteRepositoryFilter.prefix=true  -Daether.remoteRepositoryFilter.groupId=true
```

The build total time went down to 13.858 s as reported by Maven, and, all the things came
from their proper place. What happened?

First, notice how we use different local repository `flocal`. That repository contains filtering
instructions for maven-resolver:

```
flocal/.remoteRepositoryFilters/
├── groupId
│   ├── groupId-atlassian.txt
│   └── groupId-groovy-plugins-release.txt
└── prefix
    └── prefixes-central.txt

```

Second, we enabled groupId and prefix filtering.
