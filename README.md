# Pivotal Build Service 101

This project shows how to use
[Pivotal Build Service](https://content.pivotal.io/blog/pivotal-build-service-now-alpha-assembles-and-updates-containers-in-kubernetes)
to manage Docker images for your apps. Stop writing `Dockerfile`!

## How to use it?

You need a Pivotal Build Service installation, up and running.

Make sure you first run `pb api set http://yourpbs.fqdn.com`
and `pb login` to target your Pivotal Build Service instance.

Create a team:
```bash
$ pb team create dev-team
Successfully created team 'dev-team'
```

Add an user to this team:
```bash
$ pb team user add uaa_user@fqdn.com -t dev-team
```

Use an user defined in your UAA configuration.

Create file `registry-creds.yml` using `registry-creds.yml.template`, and set your parameters:
```yaml
team: dev-team
registry: harbor.fqdn.com
username: registry-user
password: registry-password
```

This file defines image registry credentials that Pivotal Build Service will use when
an app is being built for your team.

Deploy these credentials using this command:
```bash
$ pb secrets registry apply -f registry-creds.yml
Successfully created registry secret for 'harbor.fqdn.com' in team 'dev-team'
```

Create file `git-creds.yml` using `git-creds.yml.template`, and set your parameters:
```yaml
team: dev-team
repository: github.com
username: github-user
password: github-access-token
```

Just like with image registry credentials, this file defines how to access source
code for your team.

Deploy these credentials using this command:
```bash
$ pb secrets git apply -f git-creds.yml
Successfully created git secret for 'github.com' in team 'dev-team'
```

Create file `app.yml` using `app.yml.template`, and set your parameters:
```yaml
team: dev-team
source:
  git:
    url: https://github.com/alexandreroman/spring-on-k8s.git
    # Set a revision to build: a branch, a tag or a SHA-1 commit id
    revision: master
build:
  env:
  # Set Java version: default is 11.*.
  - name: BP_JAVA_VERSION
    value: 8.*
image:
  tag: harbor.fqdn.com/myrepo/spring-on-k8s
```

Create as many app definitions as you need: Pivotal Build Service will then
monitor source code updates and a Docker image will automatically be deployed
to the image registry.

Deploy this app definition using this command:
```bash
$ pb image apply -f app.yml
Successfully applied image configuration 'harbor.fqdn.com/myrepo/spring-on-k8s'
```

At this point, Pivotal Build Service knows where your app lives (your Git repo),
and how to build it to create an image (deployed to your Docker registry).

Monitor app activity using this command:
```bash
$ pb image builds harbor.fqdn.com/myrepo/spring-on-k8s
Build    Status      Image    Started Time           Finished Time    Reason
-----    ------      -----    ------------           -------------    ------
    1    BUILDING    --       2019-09-17 15:47:03    --               CONFIG
```

As you can see, the app is being built by Pivotal Build Service.
Get build logs using this command:
```bash
$ pb image logs harbor.fqdn.com/myrepo/spring-on-k8s -b 1 -f
[build-step-credential-initializer] {"level":"info","ts":1568735253.4923153,"logger":"fallback-logger","caller":"creds-init/main.go:40","msg":"Credentials initialized.","commit":"002a41a"}
[build-step-credential-initializer]
[build-step-git-source-0] git-init:main.go:81: Successfully cloned "https://github.com/alexandreroman/spring-on-k8s.git" @ "9eb9ed895e945891d1af6982b035f86c3a06ea3d" in path "/workspace"
[build-step-git-source-0]
[build-step-prepare]
[build-step-detect] Trying group 1 out of 3 with 27 buildpacks...
[build-step-detect] ======== Results ========
[build-step-detect] skip: Cloud Foundry Archive Expanding Buildpack
[build-step-detect] pass: Pivotal OpenJDK Buildpack
[build-step-detect] pass: Pivotal Build System Buildpack
[build-step-detect] pass: Cloud Foundry JVM Application Buildpack
[build-step-detect] pass: Cloud Foundry Spring Boot Buildpack
[build-step-detect] pass: Cloud Foundry Apache Tomcat Buildpack
[build-step-detect] pass: Cloud Foundry DistZip Buildpack
[build-step-detect] skip: Cloud Foundry Procfile Buildpack
[build-step-detect] skip: Pivotal AppDynamics Buildpack
[build-step-detect] skip: Pivotal AspectJ Buildpack
[build-step-detect] skip: Pivotal CA Introscope Buildpack
[build-step-detect] pass: Pivotal Client Certificate Mapper Buildpack
[build-step-detect] skip: Pivotal Elastic APM Buildpack
[build-step-detect] skip: Pivotal JaCoCo Buildpack
[build-step-detect] skip: Pivotal JProfiler Buildpack
[build-step-detect] skip: Pivotal JRebel Buildpack
```

When the app is finally built, an image will be deployed to the Docker registry:
```bash
$ pb image builds harbor.fqdn.com/myrepo/spring-on-k8s
Build    Status     Image       Started Time           Finished Time          Reason
-----    ------     -----       ------------           -------------          ------
    1    SUCCESS    e1e3e81c    2019-09-17 15:47:03    2019-09-17 15:50:50    CONFIG
```

A tag was assigned to this build: look in your Docker registry to find out.
Then, you can run this app:
```bash
docker run --rm -p 8080:8080/tcp harbor.fqdn.com/myrepo/spring-on-k8s:b1.20190917.073706
Unable to find image 'harbor.fqdn.com/myrepo/spring-on-k8s:b1.20190917.073706' locally
b1.20190917.073706: Pulling from aro/spring-on-k8s
5eac7891246b: Already exists
fbff0dd9a30d: Already exists
1fcff4a7315a: Pull complete
07a4589e264a: Pull complete
4593a41d40b2: Pull complete
defeec6a0675: Pull complete
fe61c8b64669: Pull complete
1b439d458fb2: Pull complete
87dd38f0e372: Pull complete
7a502f19c0ec: Pull complete
27ea8c71158d: Pull complete
898c18298d55: Pull complete
78b193c3b2e1: Pull complete
981ca5b9a63a: Pull complete
deb96c9efea9: Pull complete
b586528a3d08: Pull complete
Digest: sha256:98e0ece8e7e8cbc59f29966e477cb247b843bcc80dededaa67b49d5a48c19d1c
Status: Downloaded newer image for harbor.fqdn.com/myrepo/spring-on-k8s:b1.20190917.073706
Calculated JVM Memory Configuration: -XX:MaxDirectMemorySize=10M -XX:MaxMetaspaceSize=87497K -XX:ReservedCodeCacheSize=240M -Xss1M -Xmx68718877238K (Head Room: 0%, Loaded Class Count: 13034, Thread Count: 250, Total Memory: 70368744177664)

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.5.RELEASE)

2019-09-17 16:23:42.446  WARN 1 --- [           main] pertySourceApplicationContextInitializer : Skipping 'cloud' property source addition because not in a cloud
2019-09-17 16:23:42.459  WARN 1 --- [           main] nfigurationApplicationContextInitializer : Skipping reconfiguration because not in a cloud
2019-09-17 16:23:42.482  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : Starting Application on 1a2928976174 with PID 1 (/workspace/BOOT-INF/classes started by vcap in /workspace)
2019-09-17 16:23:42.485  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : No active profile set, falling back to default profiles: default
2019-09-17 16:23:45.874  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2019-09-17 16:23:47.195  INFO 1 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8080
2019-09-17 16:23:47.212  INFO 1 --- [           main] i.pivotal.demos.springonk8s.Application  : Started Application in 5.474 seconds (JVM running for 6.517)
```

As soon as you update your app source code, a new image is built
by Pivotal Build Service:
```bash
$ pb image builds harbor.fqdn.com/myrepo/spring-on-k8s
Build    Status     Image       Started Time           Finished Time          Reason
-----    ------     -----       ------------           -------------          ------
    1    SUCCESS    e1e3e81c    2019-09-17 15:47:03    2019-09-17 15:50:50    CONFIG
    2    SUCCESS    54236317    2019-09-17 16:16:14    2019-09-17 16:17:09    COMMIT
```

## Contribute

Contributions are always welcome!

Feel free to open issues & send PR.

## License

Copyright &copy; 2019 [Pivotal Software, Inc](https://pivotal.io).

This project is licensed under the [Apache Software License version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
