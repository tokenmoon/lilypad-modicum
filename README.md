## MODICUM demo

Tested on Ubuntu 22.04

Have docker, ngrok and node.js >= v16 installed.
```
snap install ngrok
sudo apt install docker.io curl python3-pip python3-virtualenv python3-dev libssl-dev
```
```
cd ~
curl -sL https://deb.nodesource.com/setup_16.x -o /tmp/nodesource_setup.sh
sudo bash /tmp/nodesource_setup.sh
sudo apt install nodejs
```

### setup block explorer

Login to https://app.tryethernal.com/settings?tab=workspace go to settings and click "RESET WORKSPACE" at the bottom.

Open a new terminal window:

```bash
snap install ngrok
ngrok http 10000
```

Copy the https url from ngrok and paste it as the RPC Server field in ethereal (in settings) then click "Update".

Then in another terminal we run the hardhat node:

```bash
git clone git@github.com:bacalhau-project/MODICUM.git
cd MODICUM/src/js
npm install --force
export ETHERNAL_EMAIL=kaiyadavenport@gmail.com
export ETHERNAL_PASSWORD=XXX
npx hardhat node --port 10000
```

Visit https://app.tryethernal.com/blocks in the browser.

IMPORTANT: each time you restart the demo - click "RESET WORKSPACE" at the bottom of the settings page on ethereal.

### various system tasks

Then create a virtualenv:

```bash
cd MODICUM/src/python/
python3 -m virtualenv venv
. venv/bin/activate
pip3 install -e .
```

From now on, activate the virtual env in any new panes where you run a python process, like this:

```bash
cd src/python
. ./venv/bin/activate
source .env
```

Then we create a new ssh keypair:

```bash
ssh-keygen -f ~/.ssh/modicum-demo
```
Hit enter three times to use the defaults

Now we adjust the values on the `src/python/.env` file paying note to the following:

 * `HOST` = `127.0.0.1`
 * `DIR` = `/home/YOURUSERNAME` (pointing to the parent directory of where you checked out MODICUM)
 * `MODICUMPATH` = `${DIR}/MODICUM`
 * `PROJECT` = `MODICUM`
 * `GETHPATH` = `/usr/local/bin`
 * `pubkey` = the public key we just generated
 * `sshkey` = the path to the private key we just generated

### influx DB

Then we setup influxDB - in another pane (install docker if you don't have it):

```bash
docker run -d \
  --name influx \
  -p 8086:8086 \
  influxdb:1.8.10
```

Now we need to setup the database:

```bash
docker exec -ti influx influx
> create database collectd;
> show databases;
exit
```

### compile contracts

Then we source the file and compile the contracts:

```bash
cd MODICUM/src/python/
source .env
echo $CONTRACTSRC
docker run -it --rm\
		--name solcTest \
		--mount type=bind,source="${CONTRACTSRC}",target=/solidity/input \
		--mount type=bind,source="${CONTRACTSRC}/output",target=/solidity/output \
		ethereum/solc:0.4.25 \
		--overwrite --bin --bin-runtime --ast --abi --asm -o /solidity/output /solidity/input/Modicum.sol
```

Ignore the warnings.

### run services

Now we start the various processes (each in it's own pane):

IMPORTANT: don't forget to activate the virtualenv in each pane!
IMPORTANT: don't forget to `source .env` in each pane!
IMPORTANT: run these in this exact order!

```bash
modicum runAsCM
```

```bash
modicum runAsSolver
```

```bash
sudo -E $(which modicum) runAsDir
```
(sudo because it will do a bunch of stuff like creating users :-O)

```bash
modicum runAsMediator
```

NOTE: replace this path with the absolute path on your system

```bash
modicum startRP --path $(realpath $PWD/../..)/0_experiments/demo/ --index 1
```

edit `0_experiments/demo/player0` to update the paths

```bash
modicum startJC --playerpath $(realpath $PWD/../..)/0_experiments/demo/ --index 0
```

Keep an eye out on the `startRP` pane - the bacalhau job ID will get printed there with a link to it on the dashboard.
