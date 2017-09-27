# Notes of running cf-abacus

CF Abacus documentation is not brilliant and it is not complete. which makes it difficult to start with. Some things are missing and others are not explained properly. It is important to understand the concepts before using it.

## References

General

 * https://github.com/cloudfoundry-incubator/cf-abacus
 * https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/README.md

It is important to understand the architecture and basics of the project:

 * https://www.cloudfoundry.org/introducing-cf-abacus-part-1-of-3
 * https://www.youtube.com/watch?v=DYYtL2xWiTc

There are really interesting reading in:

 * Auth doc: Explains how abacus uses UAA for authentication between components. Also talks about the usage of Eureka: https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/doc/security.md
 * Resource provider docs: to undestand how to providers interact with abacus:  https://github.com/cloudfoundry-incubator/cf-abacus/wiki
 * Tests doc: How to run tests. Mentions some config variables: https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/doc/tests.md
 * config doc: Ports when running local, how to configure abacus to run locally, in CF https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/doc/conf.md
 * The readme of cf applications bridge, to send real metrics of applications: https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/lib/cf/applications/README.md

There are some scripts and tools that would help to deploy and run it:

 * The bosh release to deploy abacus, contains a manifest and script to configure it: https://github.com/cloudfoundry-incubator/cf-abacus/tree/master/etc/bosh/jobs/deploy-abacus/templates
 * The concurse pipelines, contain info of how to deploy it: https://github.com/cloudfoundry-incubator/cf-abacus/tree/master/etc/concourse


## How to run it locally.

My first challenge was to run cf-abacus locally. Get actual metrics of apps usage and generate a aggregated report. Here are the steps:

### Installing dependencies

You must use node 8.1.5 and npm <5.0.0

I recommend using nvm [ https://github.com/creationix/nvm] to install node 8.1.5:

```
nvm install 8.1.5
nvm use 8.1.5
npm install -g npm@<5.0.0
```

#### Misc

Note: This project uses a command line tool called foreach. I am not sure where to get that, but I wrote my own:


```
$ cat ~/bin/foreach
#!/bin/bash

set -e -u -o pipefail
cmd=$1; shift
for i in "$@"; do
 "$cmd" "$i"
done
```


### Building and starting the cf-abacus

```
npm install
npm run build
npm start
```

Wait, all the services will be started.

### Running the demo.

I was not able to run the demo as it says in the readme. Instead I did:

```
cd node_modules/abacus-demo-client
npm install
../.bin/mocha
```

I think it is some kind of problem with motcha.

Simulate some fake usage of stuff, and creates a report.

### Sending metrics from CF apps usage using the bridge.

cf-abacus is a generic aggregator meter/accounting software. You need something that will read the events form CF (https://docs.cloudfoundry.org/running/managing-cf/usage-events.html) and send them to cf-abacus: the cf-application bridge.

To make it work, first create the corresponding UAA credentials. See  https://github.com/cloudfoundry-incubator/cf-abacus/blob/master/lib/cf/applications/README.md

```

UAA_ADMIN_PASS=.... # get this from the cf manifest.  you can get it from the bosh manifest: bosh -d cf manifest

gem install cf-uaac

UAA_API=https://uaa.$(cf api | sed -n 's/.*https:\/\/api\.\(.*\)/\1/p')

uaac target https://uaa.${UAA_API} --skip-ssl-validation
uaac token client get admin -s ${UAA_ADMIN_PASS}
uaac client add abacus-cf-applications --name abacus-cf-applications --authorized_grant_types client_credentials --authorities cloud_controller.admin --secret secret

```

Then, we must WIPE ALL THE previous usage events, aka create a billing epoch https://docs.cloudfoundry.org/running/managing-cf/usage-events.html#creating-your-billing-epoch. cf-abacus will only process any event newer than 10 minutes.

```
cf login ...
cf curl -v -d '' -X POST /v2/app_usage_events/destructively_purge_all_and_reseed_started_apps
```

Now we can run the app with:

```
cd lib/cf/applications/

npm install

API=$(cf api | sed -n 's/.*\(https:\/\/api\..*\)/\1/p')

CF_CLIENT_ID=abacus-cf-applications \
CF_CLIENT_SECRET=secret \
AUTH_SERVER=${API} \
API=${API} \
COLLECTOR=http://localhost:9080 \
DB=http://localhost:5984 \
NODE_TLS_REJECT_UNAUTHORIZED=0 \
npm start -- cf
```

You can stop with

```
npm stop -- cf
```

### Getting a report

Once the thing has been running for a while, you can generate a report doing:

```
ORG_GUID=$(cf org myorg --guid)
curl http://localhost:9088/v1/metering/organizations/${ORG_GUID}/aggregated/usage | jq .
```

### Setting log level

You can enable debugging of each component with:

```
curl localhost:9500/debug?config='*'
```


### Next steps:

 * Deploy this to CF:
   * We must vendor all the dependencies
   * We must configure the right endpoints for each component
   * We must configre the UAA roles and token verification

 * Use a couchdb DB:
   * Deploy a broker and create instances.


### Lessons learnt

 * The architecture of cf-abacus is complex to allow massive scale in Bluemix
 * Code is sparse and the documentation is not clear. Difficult to start with it.
 * You must use the very same versions to run it.
 * Important to understand the architecture.
 * The cf-abacus core does generic aggregation. The bridge sends the usage events per org.


```

```#!/bin/bash

export AWS_ACCESS_KEY_ID=minio
export AWS_SECRET_ACCESS_KEY=minio123

mkdir -p a b c

touch a/{1..1000}

export MINIO_ENDPOINT=http://minio.local:9000
set -e

aws --endpoint $MINIO_ENDPOINT \
  s3 sync --delete a s3://test-bucket/test
aws --endpoint $MINIO_ENDPOINT \
  s3 sync --delete b s3://test-bucket/test
aws --endpoint $MINIO_ENDPOINT \
  s3 sync --delete s3://test-bucket/test c

