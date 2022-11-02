# heroku-to-control-plane
Playbook for migrating from Heroku to Control Plane, controlplane.com

Adds `cpl` command to complement default CPLN cli `cpln` with "heroku style" scripting

## Instalation (atm just as local clone, not as gem)
- install CPLN cli (adds `cpln` command)
- install this repo locally, e.g.:
```sh
git clone https://github.com/shakacode/heroku-to-control-plane
```
- alias `cpl` command globally for easier access, e.g.:
```sh
# in some local shell startup scripts - .profile, .bashrc, etc.
alias cpl="~/projects/heroku-to-control-plane/cpl"
```
- project specific configs are kept in `.controlplane/` directory. `cpl` will pick those depending from which project folder tree it is executed. So, it is ok to run several projects with different configs w/o explicitly switching.

## Project changes
1. create `.controlplane` directory in your project and copy files from `templates` directory of this repo to something as following:
```sh
app_main_folder
  .controlplane
    controlplane.yaml
    Dockerfile          # this is your app Dockerfile, with some CPLN changes
    entrypoint.sh       # app specific, edit as needed
    runner.sh           # wrapper needed for `run` command, no need to edit
```

2. copy your `Dockerfile` to `.controlplane/Dockerfile` and apply following changes:
```dockerfile
# adds `netcat` package (or whatever alternative way)
RUN apt-get update && apt-get install -y netcat

# copies `entrypoint.sh` and `runner.sh` to `/app` (or whatever alternative way)
COPY .controlplane/entrypoint.sh .controlplane/runner.sh /app

# specify entrypoint script
ENTRYPOINT ["/app/entrypoint.sh"]

# run rails (or whatever alternative way)
CMD ["rails", "s"]
```
> NOTE: `netcat` and `runner.sh` are needed to enable `run` command functionality

3. edit `controlplane.yaml` where necessary, e.g.:
```yaml
# config options for one-off containers via `run` command
one_off:
  org: shakacode-staging
  location: aws-us-east-2
  workload: rails
# config options for review-apps
review_apps:
  prefix: ror-tutorial
  org: shakacode-staging
  location: aws-us-east-2
  workloads:
    - rails
    # - sidekiq
```

## Commands:

### common options
```
-a, --app XXX         app ref on CPLN (== GVC)
-w, --workload XXX    workload, where applicable
```

### `build`
- builds and deploys image to CPLN
- for simplicity review apps are build with `:latest` tag, no versioning
- force restarts workload after build (for `:latest`)

```sh
cpl build -a ror-tutorial
```


### `config`
- display current configs (global and project specific)

```sh
cpl config
```

### `logs`
- light wrapper to display tailed raw logs for app/workload syntax

```sh
# display logs for default workload (== one_off.workload)
cpl logs -a ror-tutorial

# display logs for other workload
cpl logs -a ror-tutorial -w postgres
```

### `run`
- runs one-off replicas (analogue of `heroku run`)
- limitation: atm, there is no way to exectute command with arbitrary args via native `cpln` cli, so it uses workaround with `runner.sh` and injected env var

```sh
# opens shell (bash by default)
cpl run -a ror-tutorial

# runs commmand, displays output, quits (as command quits)
cpl run ls / -a ror-tutorial
cpl run rails db:migrate:status -a ror-tutorial

# runs command, keeps shell opened
cpl run rails c -a ror-tutorial
```

### `setup`
- applies app specific configs to general templates (e.g. for every review-app)
- publishes (creates/updates) those at CPLN
```sh
# applies single template
cpl setup redis -a ror-tutorial

# applies several templates (practically creating full app)
cpl setup gvc postgres redis rails -a ror-tutorial
```
- template variables
```
APP_GVC      - basically gvc or app name
APP_LOCATION - default location
APP_ORG      - org
APP_IMAGE    - image
```
