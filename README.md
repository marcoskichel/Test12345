# hg-api

Core API for the Hugo backend.

[![CircleCI](https://circleci.com/gh/hugoai/hg-api/tree/master.svg?style=svg&circle-token=ac9b59684f51f0c7f9bcfceea2d1cd479307c596)](https://circleci.com/gh/hugoai/hg-api/tree/master)
[![codecov](https://codecov.io/gh/hugoai/hg-api/branch/master/graph/badge.svg?token=ZERIsYoqaD)](https://codecov.io/gh/hugoai/hg-api)

### How do I get set up?

1. `git clone git@github.com:hugoai/hg-api.git`
2. `yarn && yarn install --dev` - Installs dependencies
3. `yarn start`

### How do I set it up (with docker)?

1. [Install docker](https://docs.docker.com/docker-for-mac/install/)
2. `git clone git@github.com:hugoai/hg-api.git`
3. `docker build -t api .`
4. `docker run api`
5. Connect to the service at `http://127.0.0.1:3000`

### How do I run tests?

-   yarn test
-   yarn lint

### How do I debug dev, stage or prod?

You should have been issued with a personal, private key and this is installed on every ECS EC2 instance.

**EC2 Instances**

1. Go to [this page](https://us-west-2.console.aws.amazon.com/ecs/home?region=us-west-2#/clusters), choose the appropriate env cluster.
2. Then go to the `ECS Instances` tab, choose any row and click on the link in the `EC2 Instance` column.
3. In the detail section find the `Public DNS (IPv4)`, copy the value to the clipboard.
4. In a terminal run `ssh -i <path to ssh private key> ec2-user@<public dns address>`. eg. `ssh -i ~/alex.pem ec2-user@ec2-54-71-208-156.us-west-2.compute.amazonaws.com`
5. You should be logged in, run `sudo su` if you need `root` access.

Common tasks:

-   Logs are at `/var/log`.
-   Check disk space using `df -h` or `du -sh /`.
-   Run `docker ps -a` to find currently running containers on that instance.

**Redis**

You should be able to access the Redis command line interface by SSH to the EC2 instance (see below).
You also need to install and run Redis locally to be able to run the API.

Common tasks:

-   To clear the Redis cache, run `redis-cli -h <redis-hostname> -c "flushdb"` and redeploy the project.
-   To install Redis locally on macOS, you can use homebrew: `brew install redis`. You can also homebrew to launch it: `brew services start redis`.

**Databases**

1. Download & install [Postico](https://eggerapps.at/postico/) (or something similar like pgAdmin).
2. Add a new "Favorite", for the basic info use:
    - Find the DB URL from the appropriate [env cfg file](https://github.com/hugoai/hg-api/tree/master/ci/env).
    - Find the appropriate user from [this list](https://github.com/hugoai/hg-secrets/blob/master/README.md)
3. Now to actually access the DB we'll SSH tunnel via the Bastion. Hit `Options -> Connect via SSH`.
    - SSH Host: `bastion.<env>.hugoai.com`
    - User: `ec2-user`. No password
    - Private key is the one you should have been issued, if you don't have one shout.
4. Hit connect and enjoy.

Tips:

-   **Be careful!** Particularly please wrap writes in transactions, restores are painful.

**Firebase**

To debug firebase functions, you should be able to use firebase CLI.
Under /functions directory run: `firebase deploy --only functions --project <project>`

**Metrics & Logs**

We use AWS CloudWatch for this primarily:

-   For metrics, see [here](<https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#metricsV2:graph=~();search=api>).
    -   We store OS (CPU, memory, etc), container (# running, etc) and service (uptime %, etc) level data.
-   For logs, see [here](https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#logs:prefix=api)
    -   We store application logs grouped by service (eg. "api-dev"). To debug an individual container instance delve into the group.

**Dashboards**

-   [Geckoboard](https://app.geckoboard.com/) - Ask for an invite to the team.

### How do I run migrations?

1. Generate a new migration file by running `yarn create-migration`. It will appear in the `/migrations` folder.
2. Amend the `up()` function to do what you need. We use [Sequelize](http://docs.sequelizejs.com/manual/tutorial/migrations.html) and [Umzug](https://github.com/sequelize/umzug).
3. Then run `yarn migrate` to run the migration. You're done.

If you ever need to revert a migration run `yarn undo-migration`. Other useful commands are:

-   `yarn reset-db` - Resets the `api` DB locally.
-   `yarn reset-db-tests` - Resets the `api_test` DB locally.
-   `yarn promote-users` - Promotes all users to admins, useful for testing [hg-accounts-ui](https://github.com/hugoai/hg-accounts-ui).

### How do I develop against integrations that require incoming traffic or HTTPS callbacks?

Basically, all you need to do is to connect to the VPN hosted at `vpn.local.hugoai.com`.

The VPN server proxies all HTTPS/HTTP traffic from vpn.local.hugoai.com to the first client that is
connected to the VPN (192.168.42.10:3000). If you're connected with a different local IP address,
it means someone else is connected, which is a limitation of this approach. You can either ask them
to disconnect or simply restart the server in AWS. 🙃

#### How to connect to the VPN?

###### MacOS:

1. Go to System Preferences and then Network.
2. Click the + button to create a new connection.
3. Select "VPN" for the interface and "L2TP over IPSec" for VPN type.
4. Use `vpn.local.hugoai.com` for the server address and `admin` for the account name.
5. Click on Authentication Settings and enter the password and shared secret. (You can find it in our 1Password Eng vault).
6. Once you're connected you should be ready to go. You can test by visiting https://vpn.local.hugoai.com and see if you get the `Hello World` message.

###### Linux:

If you're on Linux, you may also need to add these:

-   Phase1 Algorithms: `aes-sha1-modp1024`
-   Phase2 Algorithms: `aes-sha1`

###### Windows:

On windows you will need to add some configuration to the network adapter, [here](https://www.snel.com/support/learn-how-to-connect-l2tp-ipsec-vpn-on-windows-10/) is a tutorial that can be followed, the VPN password and shared key can be found under the engeneering vault at [1Password](https://hugo-team.1password.com/).

If you are using `WSL2`, you may need to configure so that the network `MTU` is the same in both windows and linux subsystem, to achieve that follow these steps:

1. In `powershell`, run the following:
   `netsh interface ipv4 show subinterface`
   Then grab the MTU value of the VPN network adapter.
2. In wsl `bash`, set the MTU value to the same number:
   `sudo ip link set eth0 mtu <MTU value>`

**Observation for Office 365**:
There's an additional step for Office 365. The Microsoft authentication uses cookies, so when we redirect the browser to our API, which in turn redirects to Office, the callback URL must match the origin URL. To do that, you have to replace the base URL in `http://local.hugoai.com:3000/auth/office365?mixpanelDistinct...` by `http://vpn.local.hugoai.com/auth/office365`, and you should be all set.

## Integrations

Where do I see the Hugo apps that were submitted to the 3rd party services?

1. Office 365: https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps

### Jira and Confluence

To enable Jira we need to use Demo Dave's Jira account (demo@testcorp.co)

Start VPN

Start the API like this: APP_URL=https://vpn.local.hugoai.com yarn start

Verify if the "Callback URL" is set in the app at https://developer.atlassian.com/apps
