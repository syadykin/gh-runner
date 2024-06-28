Self-hosted GitHub actions runner image
=======================================

This image is intended to run a self-hosted GitHub actions runner in the docker.

This runner registers and de-registers on the GitHub organization level automatically, for this purpose two environment variables should be passed during run time:

* `GH_OWNER` - organization slug
* `GH_TOKEN` - token with permission to register a new runner on the organizational level (see [link](https://docs.github.com/en/rest/actions/self-hosted-runners?apiVersion=2022-11-28#create-a-registration-token-for-an-organization) for details)

Usage is pretty simple:

```
$ docker pull syadykin/gh-runner
$ docker run -d --rm \
  --privileged \
  --name my-runner \
  -e GH_OWNER=my-org \
  -e GH_TOKEN=github_pat_1....... \
  -e RUNNER_NAME=runner \
  syadykin/gh-runner:latest
```
Please note that this is the `one-shot` runner and will exit after job completion (or fail) to clean up all the changes made, so it is better to run it with systemd with parameter support:

`gh-runner@.service` file content:

```
[Unit]
Description=GitHub Runner Service runner-%i
Requires=docker.service
After=docker.service

[Service]
Type=simple
Restart=always
EnvironmentFile=/etc/default/gh-runner
ExecStart=/bin/bash -c "docker run --privileged --name runner-%i --rm -e GH_OWNER=${GH_OWNER} -e GH_TOKEN=${GH_TOKEN} -e RUNNER_NAME=runner-%i ${ARGS} syadykin/gh-runner:latest"

[Install]
WantedBy=multi-user.target
```

The `/etc/defaults/gh-runner` file should have at least `GH_OWNER` and `GH_TOKEN` defined:

```
GH_OWNER=my-org
GH_TOKEN=github_pat_1.......
ARGS=
```

Enable runner:

```
$ systemctl daemon-reload
$ systemctl enable gh-runner@alpha
$ systemctl start gh-runner@alpha
```

Start the second runner:

```
$ systemctl enable gh-runner@beta
$ systemctl start gh-runner@beta
```

In addition, you can pass certain parameters to docker run, which should be placed into the `ARGS` variable:

```
ARGS=-v /etc/gh-runner/env:/home/runner/.env
```

In this case file `/etc/gh-runner/env` may have any additional environment variables defined, which become available in the runner job. So if you use this runner with AWS, it's a great place to put the AWS key and secret:

```
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=DrvB...
```

Happy building!

Changelog
--------

* 0.1.0 - switch to Ubuntu 24.04

* 0.1.1 - update gh runner to 2.317.0
