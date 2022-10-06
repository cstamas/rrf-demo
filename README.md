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
HTTP requests are issued only to get 404 response), but also "leaks" your dependencies to
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
Not that its good thing, the "nasty project" issues should be fixed after all (at least restore proper
repository ordering, making Maven Central first), but this demo just shows how powerful is filtering.

The instructions for remote repository filtering are following:
* `prefix/prefixes-central.txt` - contains the list of contained prefixes in Maven Central.
* `groupId/groupId-atlassian.txt` - the list of ALLOWED `groupId`s from `atlassian` remote repository.
* `groupId/groupId-groovy-plugins-release.txt` - the list of ALLOWED `groupId`s from `groovy-plugins-release` remote repository.

Two filter implementations together fixed all the problems.

### Prefixes filter

The prefixes filter relies on file containing list of "repository prefixes" available from given repository. 
The prefix is essentially "starts with" of Artifact path as translated by Repository Layout. Its effect is that 
only those artifacts will be attempted to be downloaded from given remote repository, if there is a 
"starts with" match between artifact path translated by layout, and prefixes file published by remote repository.

Example: `com.corp.theproject:api:1.0` artifact, when translated to path by default layout results in
`com/corp/theproject/api/1.0/api-1.0.jar` path. So the following prefixes may be used to "allow" this artifact 
(and more!):

* `/com/corp/theproject/api/1.0` - would allow only 1.0 version of this Artifact
* `/com/corp/theproject/api` - would allow any version of this Artifact 
* `/com/corp/theproject/` - would allow any artifactId and version of this Artifact
* `/com/corp/` - would allow whole groupId of this artifact

Prefixes are path prefixes, hence, are kinda filtering other way around, 
is rather remote repository advising us "do not even bother by coming to me with a path that has no 
appropriate prefix enlisted in this file". On the other hand, having a prefix enlisted does not
provides 100% guarantee that matched artifact is really present! For example presence of `/com/foo`
prefix does NOT implies that `com.foo:baz:1.0` artifact is present, it merely tells I do have
something that starts with `/com/foo` (for example `com.foo.baz:lib:1.0`). The depth of published 
prefixes is usually set by publisher, and is usually 2-3. It all boils down to equilibrium of 
"best coverage" and "prefixes file size" (ultimately, prefixes file containing all the relative
paths of deployed artifact from repository root would be 100%, but the cost would be huge
file size for repositories like Maven Central).

As this file is (automatically) published by MC and MRMs, using them is simplest. Manual authoring 
of these files, while possible, is not recommended. Best is to keep them up to date by 
downloading published files from remote repositories.

Many MRMs and Maven Central itself publishes this file. Some prefixes file examples:
* Maven Central [prefixes.txt](https://repo.maven.apache.org/maven2/.meta/prefixes.txt)
* ASF Releases hosted repository [prefixes.txt](https://repository.apache.org/content/repositories/releases/.meta/prefixes.txt)

The prefixes files are expected in following location by default: `${localRepo}/.remoteRepositoryFilters/prefix/prefixes-${remoteRepository.id}.txt`.

In this example project I just used `curl` to get the `prefixes.txt` as published by Maven Central
and placed it at expected location under `flocal` local repository.

### GroupIds filter

The other implementation is filtering based on allowed `groupId` of Artifact. In essence, is a list
of "allowed groupId coordinates from given remote repository".

The groupId files are expected in following location by default: `${localRepo}/.remoteRepositoryFilters/groupId/groupId-${remoteRepository.id}.txt`.

In this example I added two filters, for two "extra" remote repositories.

The `groovy-plugins-release` is allowed only for one groupId:
* `org.codehaus.groovy` as both the artifacts are from this group along will all bits like parent POMs.

The `atlassian` repository is allowed for 3 groupIds:
* `com.atlassian.audit` - the main dependency of example project.
* `com.atlassian.platform` - dependency or parent or import POM.
* `com.atlassian.pom` - parent POM, I guess.

From these two remote repositories am not interested in anything else.
