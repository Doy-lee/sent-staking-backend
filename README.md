# SENT Staking Backend

## Running the backend

### Pre-requisites

    apt install build-essential python3-dev python3-flask

    python3 -m pip install eth_utils web3

To run the backend on **Ubuntu >= 24.04**:
```shell
apt install build-essential python3-pip python3-dev pybind11-dev liboxenc-dev liboxenmq-dev python3.12-venv
pip install eth_utils web3 PyNaCl Flask uWSGI
```

Additionally, you need:

**Python bindings for oxen-mq & oxen-encoding**

Instructions available at:

- [oxen-pyoxenc](https://github.com/oxen-io/oxen-pyoxenc)
- [oxen-pyoxenmq](https://github.com/oxen-io/oxen-pyoxenmq)

These packages require the development headers by setting up the
[Oxen Deb Repository](https://deb.oxen.io). Follow those instructions and then
install the following packages

    apt install liboxenc-dev liboxenmq-dev

Before running an instance, `oxend` must be running and its address/smart contracts configured in
`config.py`.

### Instance

It's possible to run the service in flask directly but the timers to poll the smart contracts
requires WSGI. Both methods are detailed below:

    FLASK_APP=sent flask run --reload --debugger
    uwsgi --http 127.0.0.1:5000 --master -p 4 -w sent --callable app

You may optionally append `--fs-reload sent.py` to the `uwsgi` invocation to
automatically restart the server when `sent.py` is modified.

After the server is running, visit `127.0.0.1:5000/info` to verify that the server is up and
responding correctly with a payload like the following:

```json
{
  "network": {
    "hard_fork": 21,
    "height": 837,
    "max_stakers": 10,
    "min_operator_stake": 30000000000,
    "nettype": "stagenet",
    "staking_requirement": 120000000000,
    "top_block_hash": "aa32d153fa260a76bd31751bc3b3193565c8cc17ec7ce543585b96d206b37892",
    "version": "11"
  },
  "t": 1720677228.877856
}
```

## Setting up an oxend instance

The default configuration for mainnet.py, testnet.py, devnet.py look for mainnet.sock, testnet.sock,
and devnet.sock in this directory, which should be either sockets or symlinks to sockets to a
running mainnet/testnet/devnet oxend.

There are a few ways to make this work.

### Symlinks

You can add symlinks here to existing oxend sockets.  If oxend and the backend code are running as
the same user then you can simply create a symlink:

    ln -s /var/lib/oxend/oxend.sock mainnet.sock

For a system service oxend (as installed in the debs, via the `oxen-node.service` systemd service),
you can make it work, but will need an extra step to configure socket permissions:

    ln -s ~/.oxen/oxend.sock mainnet.sock

and then do one of:

- Set the active group of the running sent staking backend to the `_loki` user.  Whether this is
  easy or not depends on how the backend service is running.

- Add the `_loki` group to the supplemental groups of the user that will be running this backend.
  (This is the more secure option).

      usermod -aG _loki FRONTENDUSERNAME

- Add the `lmq-umask=0000` option to `oxen.conf` (or run oxend with the `--lmq-umask=0000` option).
  Note that this is somewhat less secure than the above, because *any* user or process on the local
  system will be able to control oxend.

Do *NOT*:

- Run any production service as root (including under sudo).  Don't be tempted just because "it
  works" under sudo: by running things under root/sudo you compromise the security of your entire
  system as a solution to properly setting up permissions.  Please don't do this, ever.

### Make oxend create the socket

The `lmq-public=ipc:///path/to/sent-staking-backend/oxend/mainnet.sock` can be added to oxen.conf
(or run with `--lmq-public=...`) to have it listen on that socket.  Note that when oxend and
sent-staking-backend are running as separate users, this has the same permission issues as the
Symlinks approach (see above for solutions).

### Use TCP OMQ RPC

You can configure oxend with `lmq-public=tcp://127.0.0.1:6789` (choose whatever port you like in
place of `6789`) in the oxen.conf config file [alternatively: run oxend with
`--lmq-public=tcp://127.0.0.1:6789`] and then add/uncomment the `mainnet_rpc=...` (or `testnet_rpc=`
or `devnet_rpc=`) line in config.py.
