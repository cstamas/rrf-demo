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

## What is happening?

You will notice that all your dependencies are downloaded from Atlassian repository (but same
would happen with Groovy repository, if that would be the first). Reason is pretty much straightforward: 
**both these repositories are "group" repositories**, that contain Maven Central proxy among
their members as well, hence, all artifacts you expect from MC can be obtained from these
repositories as well! As Maven goes "round robin" on ordered list of remote repositories, it will
find everything in first, in this case, Atlassian repository. Hence, as an side-effect, you are basically getting 
MC Artifacts via Atlassian infrastructure, or Groovy repository if that would be first (as same stands for that as well). 
Not only is slower, but has a nice dangerous side as well.

Moreover, we might skim over the fact, that Groovy compiler artifact, present only in `groovy-plugins-release`
remote repository were asked for from Atlassian as well. This "leakage" of artifact requests is caused
by the fact that Maven goes "round robin" just to get the needed artifact, by iterating (in
effective POM order) thru remote repositories. This not only adds extra time to build (as 
HTTP requests are issues only to get 404 response), but also "leaks" your dependencies to
those repositories that for sure have no such thing.

## Lets fix this

This is where Remote Repository Filtering comes to play. We can fix this ONLY by filtering
(while this "nasty" project does several things wrongly, we will NOT touch any of the POM or
custom settings XML to achieve this fix).

For this, you need following things:

* local build of maven-resolver PR https://github.com/apache/maven-resolver/pull/197
* local build of maven-3.9.x branch (w/ change to use local maven-resolver build)
* deployed the custom Maven build somewhere

After all that above, let's build this nasty project using following commands (assuming
you unpacked custom Maven build to ~/tmp/apache-maven-3.9.0-SNAPSHOT directory). Observe 
that this time we use `flocal` local repository, is important bit:

```
$ ~/tmp/apache-maven-3.9.0-SNAPSHOT/bin/mvn -s settings.xml -Dmaven.repo.local=flocal clean package \
  -Daether.remoteRepositoryFilter.prefix=true  -Daether.remoteRepositoryFilter.groupId=true
```

The build total time went down to 13.858 s as reported by Maven, and all the things came
from their proper and expected origin. This was RRF in action.

## What is happening? (part 2)

First, notice how we used different local repository `flocal`. That repository contains filtering
instructions for maven-resolver:

```
flocal/.remoteRepositoryFilters/
├── groupId
│   ├── groupId-atlassian.txt
│   └── groupId-groovy-plugins-release.txt
└── prefix
    └── prefixes-central.txt
```

Second, we enabled two kind of filtering: `groupId` and `prefix`. No POM or settings.xml was changed.
Not that is good thing, the "nasty project" issues should be fixed after all (at least restore proper
repository ordering, making Maven Central), but this demo just shows for powerful is filtering.

The instructions for remote repository filtering are following:
* prefix/prefixes-central.txt - contains the list of contained prefixes in Maven Central.
* groupId/groupId-atlassian.txt - the list of ALLOWED `groupId`s from `atlassian` remote repository.
* groupId/groupId-groovy-plugins-release.txt - the list of ALLOWED `groupId`s from `groovy-plugins-release` remote repository.

Two filter implementations together fixed all issues.

### Prefixes filter

The prefixes filter relies on file containing list of "repository prefixes" available from given repository. 
The prefix is essentially "starts with" of Artifact path as translated by Repository Layout. It results that 
remote repository NOT having some prefix, will not be attempted to fetch artifact having path of missing 
prefix. Or in other words, only those artifacts will be downloaded from given remote repository, if there
is a "starts with" match between artifact path translated by layout, and prefixes file published by 
remote repository.

Example: `com.corp.theproject:api:1.0` artifact, when translated to path by default layout results in
`com/corp/theproject/api/1.0/api-1.0.jar` path. So the following prefixes may be used to "allow" this artifact 
(and more!):

* `/com/corp/theproject/api/1.0` - would allow only 1.0 version of this Artifact
* `/com/corp/theproject/api` - would allow any version of this Artifact 
* `/com/corp/theproject/` - would allow any artifactId and version of this Artifact
* `/com/corp/` - would allow whole groupId of this artifact

Prefixes are path prefixes, hence, are kinda filtering other way around, 
is rather remote repository advising us "do not even bother by coming to me with a path that has no 
appropriate prefix enlisted in this file". Also, as this file is (automatically) published by
MC and MRMs, using them is simplest. Manual authoring of these files, while possible, is not 
recommended. Best is to keep the up to date by downloading as published by remote repositories.

Important: As filtering is "starts with", with prefix `/com/foo` you not enabled only `com.foo:bar.1.0`
artifact but also `com.foo.bar:baz:1.0` and `com.foo:bar-baz:1.0` and so on. It is important to
understand that this filter is more like a statement from far end, than you as client limiting
what to get from it.

Many MRMs and Maven Central itself publishes this file. Some prefixes file examples:
* [Maven Central](https://repo.maven.apache.org/maven2/.meta/prefixes.txt)
* [ASF Releases](https://repository.apache.org/content/repositories/releases/.meta/prefixes.txt)

The prefixes files are expected in following location by default: `${localRepo}/prefix/prefixes-${remoteRepository.id}.txt`.

### GroupIds filter

The other implementation is filtering based on allowed groupId of Artifact coordinate. In essence, is list
of "allowed groupId coordinates from given remote repository".

The groupId files are expected in following location by default: `${localRepo}/groupId/groupId-${remoteRepository.id}.txt`.

