# The Nasty Project

This is an example of a "nasty" project that does all wrong.

It is a very basic project, that uses `groovy-eclipse-compiler` to compile
sources, and for example sake, it depends on Atlassian Audit API artifact 
`com.atlassian.audit:atlassian-audit-api`, none of these are available from Maven Central.

As these two are not present on Maven Central, extra repositories needs to be
added to project, and to make this example "even worse", this project 
uses custom `settings.xml` to achieve that.

The extra repositories are set up as explained in corresponding documentations:
* Groovy Eclipse Compiler: https://github.com/groovy/groovy-eclipse/wiki/Groovy-Eclipse-Maven-plugin
* Atlassian: https://developer.atlassian.com/server/framework/atlassian-sdk/atlassian-maven-repositories-2818705/

To build this "nasty" project, do this (use pristine local repository, to force Maven download
all the needed artifacts). and **use Maven 3.8.x**:

```
$ rm -R local-repo   <<< if running subsequent time
$ mvn -V -s settings.xml package
```

The **build time was 2:28 minutes** as reported by Maven 3.8.x. But, just take a peek at full Maven console
output... (sit down before doing it).

Now, nuke local repository (`rm -R repo-local`) and **use Maven 3.9.3** to build same project:

```
$ rm -R local-repo   <<< if running subsequent time
$ mvn -V -s settings.xml clean package
```

The **build time was 0:20 seconds** as reported by Maven 3.9.x.

## What is happening?

You will notice that most if not all of your plugins and dependencies are downloaded 
from Atlassian and Groovy repository! Reason is pretty much straightforward: 
**both these repositories are "group" repositories**, that contain Maven Central proxy among
their members as well, hence, all artifacts you expect from MC can be obtained from these
repositories as well! As Maven goes "loop" on ordered list of remote repositories, it will
find everything in first repository. Hence, as an side-effect, you are basically getting 
MC Artifacts via Groovy or Atlassian infrastructure, something you should not do. 
This is not only slower, but has a peculiar danger edge as well.

Moreover, we can notice that Groovy compiler artifact, **present only** in `groovy-plugins-release`
remote repository may be asked for from Atlassian as well. This "leakage" of artifact requests is caused
again by the fact of looping repositoeies of Maven, by iterating (in
effective POM order) thru remote repositories. This not only adds extra time to build (as 
HTTP requests are issued only to get 404 response), but also "leaks" your dependencies to
those repository maintainers as well (they will have all these in their access logs).

Conclusion: not only build time is unacceptable, but you got artifacts that are hosted on
Maven Central (CDN backed) from other places instead, this is a mess.

(note: the rrf-demo project is changing, so things may not happen exactly as above, but the
point is leakage: Maven asks for things from multiple repositories, and those does not have
it will still be asked for -- leaks requests)

## Lets fix this

This is where Remote Repository Filtering comes to play. **We can fix this solely by filtering**
(while this "nasty" project does several things wrongly, we will NOT touch any of the POM or
custom settings XML to achieve this fix).

Maven 3.9.3 used resolver contains "remote repository filtering" (RRF) implementation.

## What is happening? (part 2)

First, notice how we used different local repository `flocal`. That repository contains filtering
instructions for maven-resolver:

```
.mvn/rrf/
├── groupId-atlassian.txt
├── groupId-groovy-plugins-release.txt
└── prefixes-central.txt
```

Second, we enabled two kind of filtering: `groupId` and `prefix`. No POM or settings.xml was changed.
Not that its good thing, the "nasty project" issues should be fixed after all (at least restore proper
repository ordering, making Maven Central first), but this demo just shows how powerful is filtering.

The instructions for remote repository filtering are following:
* `prefixes-central.txt` - contains the list of ALLOWED prefixes in Maven Central.
* `groupId-atlassian.txt` - the list of ALLOWED `groupId`s from `atlassian` remote repository.
* `groupId-groovy-plugins-release.txt` - the list of ALLOWED `groupId`s from `groovy-plugins-release` remote repository.

Two filter implementations together fixed all the problems.

### Prefixes filter

The prefixes filter relies on file containing list of "repository prefixes" available from given repository. 
The prefix is essentially "starts with" of Artifact path as translated by Repository Layout. Its effect is that 
only those artifacts will be attempted to be downloaded from given remote repository, if there is a 
"starts with" match between artifact path translated by layout, and prefixes file published by remote repository.

Prefixes are usually published by remote repositories, hence, are kinda filtering other way around: 
is rather remote repository advising us "do not even bother by coming to me with a path that has no 
appropriate prefix enlisted in this file". On the other hand, having a prefix enlisted does not
provides 100% guarantee that matched artifact is really present! For example presence of `/com/foo`
prefix does NOT implies that `com.foo:baz:1.0` artifact is present, it merely tells "I do have
something that starts with `/com/foo`" (for example `com.foo.baz:lib:1.0`). The depth of published 
prefixes is usually set by publisher, and is usually 2-3 or 4. It all boils down to equilibrium of 
"best coverage" and "file size" (ultimately, prefixes file containing all the relative
paths of deployed artifact from repository root would be 100% coverage, but the cost would be huge
file size for huge repositories like Maven Central).

As this file is (automatically) published by MC and MRMs, using them is simplest. Manual authoring 
of these files, while possible, is not recommended. Best is to keep them up to date by 
downloading published files from remote repositories.

Many MRMs and Maven Central itself publishes this file. Some prefixes file examples:
* Maven Central [prefixes.txt](https://repo.maven.apache.org/maven2/.meta/prefixes.txt)
* ASF Releases hosted repository [prefixes.txt](https://repository.apache.org/content/repositories/releases/.meta/prefixes.txt)

The prefixes files are expected in following location by default: `${localRepo}/.remoteRepositoryFilters/prefixes-${remoteRepository.id}.txt`.

In this example project I just used `curl` to get the `prefixes.txt` as published by Maven Central
and placed it at expected location under `flocal` local repository.

### GroupIds filter

The other implementation is filtering based on allowed `groupId` of Artifact. In essence, is a list
of "allowed groupId coordinates from given remote repository".

The groupId files are expected in following location by default: `${localRepo}/.remoteRepositoryFilters/groupId-${remoteRepository.id}.txt`.

In this example I added two filters, for two "extra" remote repositories.

The `groovy-plugins-release` is allowed only for one groupId:
* `org.codehaus.groovy` as both the artifacts are from this group (along will all bits like parent POMs, I guess).

The `atlassian` repository is allowed for 3 groupIds:
* `com.atlassian.audit` - the main dependency of example project.
* `com.atlassian.platform` - dependency or parent or import POM, I guess.
* `com.atlassian.pom` - dependency or parent or import POM, I guess.

From these two remote repositories am not interested in anything else.

The list of allowed groupIds I assembled by "trial and error". The two groupIds for explicit
dependencies I knew, then tried the build, and observed what other groupId are required. The Groovy 
was "satisfied" with this 1 entry, while for Atlassian I had 3 iterations to collect the 3 "allowed"
groupIds.

The GroupId filter allows "recording" of encountered groupIds as well, that can be used as
starting point: after "recording" done, one can edit it, remove or add entries as needed. When
groupId filter set to "record", it does NOT filter, but instead collects all the encountered 
groupIds per remote repository and saves them into properly placed file(s).

## But how?

The RRF currently have two filter implementations: prefix and groupId. To make these filters operate, you 
have to enable them. In example both were enabled, but there is an important detail: for remote repository 
not having filter input present, the filter pulls out from "voting"
(does not participate in filtering). 

Hence, we created following situation:

| Remote Repository      |   Prefix Filter  | GroupId Filter |
|------------------------|------------------|----------------|
| central                | active           | inactive       |
| groovy-plugins-release | inactive         | active         |
| atlassian              | inactive         | active         |

It resulted in following "constraints":
* Maven Central asked only for those artifacts it claims it have (prefixes)
* groovy-plugins-release and atlassian were asked only for allowed groupIds.

In short, enabling filters are not enough, to make the active for a remote repository, you
must provide them "input data" for given remote repository as well.

### Remarks

General note: the "build time" is presented only for demo purposes (and makes sense only to compare them
on same computer, as relative numbers). For those curious, this test was done on a Linux laptop using
WiFi, and using direct repository URLs, no proxies/caches/MRMs involved.

