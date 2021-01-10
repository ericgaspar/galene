# Installation

## Build the server binary

    CGO_ENABLED=0 go build -ldflags='-s -w'

## Create a server certificate

    mkdir data
    openssl req -newkey rsa:2048 -nodes -keyout data/key.pem -x509 -days 365 -out data/cert.pem

## Set the server administrator credentials

This step is optional.

    echo 'god:topsecret' > data/passwd

## Set up an ICE server

ICE is the NAT and firewall traversal protocol used by WebRTC.  ICE uses
a variety of techniques for establishing a flow in the presence of
a firewall; the two most effective techniques, STUN and TURN, require help
from an external server.  Whether you need a helping server depends both
on your firewalling setup and on the networks of your users; for
production use, you should probably use your own TURN server.

### No ICE server

If Galène is not firewalled (high-numbered ports are accessible from the
Internet) and none of your users are on a restrictive network, then you
need no ICE servers.  There is nothing to do, skip to *Set up a group*
below.

### STUN server

If Galène might be behind a firewall (high-numbered ports might or might
not be accessible from the Internet), but none of your clients are on
a restrictive network, then a STUN server is enough.  It is usually safe
to use a third-party STUN server, although doing that might violate the
privacy of your users.  Your `data/ice-servers.json` file should look like
this:

    [
        {
            "urls": [
                "stun:stun.example.org"
            ]
        }
    ]
    
### TURN server

In practice, some of your users will be on restrictive networks: many
enterprise networks only allow outgoing TCP to ports 80 and 443;
university networks tend to additionally allow outgoing traffic to port
1194.  For best performance, your TURN server should be located close to
Galène and close to your users, so you will want to run your own (I use
*coturn*, but other implementations of TURN should work too).

Your `ice-servers.json` should look like this, where `username` and
`secret` are identical to the ones in your TURN server's configuration:

    [
        {
            "urls": [
                "turn:turn.example.org:443",
                "turn:turn.example.org:443?transport=tcp"
            ],
            "username": "galene",
            "credential": "secret"
        }
    ]

If you use coturn's `use-auth-secret` option, then your `ice-servers.json`
should look like this:

    [
        {
            "urls": [
                "turn:turn.example.com:443",
                "turn:turn.example.com:443?transport=tcp"
            ],
            "username": "galene",
            "credential": "secret",
            "credentialType": "hmac-sha1"
        }
    ]
    
For redundancy, you may set up multiple TURN servers, and ICE will use the
first one that works.

## Set up a group

A group is set up by creating a file `groups/name.json`.

    mkdir groups
    vi groups/groupname.json
    
A group with a single operator and no password for ordinary users looks
like this:

    {
        "op": [{"username": "jch", "password": "1234"}],
        "presenter": [{}]
    }
   
A group with one operator and two users looks like this:

    {
        "op": [{"username": "jch", "password": "1234"}],
        "presenter": [
            {"username": "mom", "password": "0000"},
            {
                "username": "dad",
                "password": "Pójdźże, kiń tę chmurność w głąb flaszy!"
            }
        ]
    }
    
More options are described under *Details of group definitions* below.

## Test locally

    ./galene &
    
You should be able to access Galène at `https://localhost:8443`.

## Deploy to your server

Set up a user *galene* on your server, then do:

    rsync -a galene static data groups galene@server.example.org:
    
Now run the binary on the server:

    ssh galene@server.example.org
    ulimit -n 65536
    nohup ./galene &

If you are using *runit*, use a script like the following:

    #!/bin/sh
    exec 2>&1
    cd ~galene
    ulimit -n 65536
    exec setuidgid galene ./galene

If you are using *systemd*:

    [Unit]
    Description=Galene
    After=network.target

    [Service]
    Type=simple
    WorkingDirectory=/home/galene
    User=galene
    Group=galene
    ExecStart=/home/galene/galene
    LimitNOFILE=65536

    [Install]
    WantedBy=multi-user.target

# Usage

## Locations

There is a landing page at the root of the server.  It contains a form
for typing the name of a group, and a clickable list of public groups.

Groups are available under `/group/groupname`.  You may share this URL
with others, there is no need to go through the landing page.

Recordings can be accessed under `/recordings/groupname`.  This is only
available to the administrator of the group.

Some statistics are available under `/stats`.  This is only available to
the server administrator.

## Side menu

There is a menu on the right of the user interface.  This allows choosing
the camera and microphone and setting the video throughput.  The
*Blackboard mode* checkbox increases resolution and sacrifices framerate
in favour of image quality.  The *Play local file* dialog allows streaming
a video from a local file.

## Commands

Typing a line starting with a slash `/` in the chat dialogue causes
a command to be sent to the server.  Type `/help` to get the list of
available commands; the output depends on whether you are an operator or
not.


# Details of group definitions

Groups are defined by files in the `./groups` directory (this may be
configured by the `-groups` command-line option, try `./galene -help`).
The definition for the group called *groupname* is in the file
`groups/groupname.json` and does not contain the group name, which makes
it easy to copy or link group definitions.  You may use subdirectories:
a file `groups/teaching/networking.json` defines a group called
*teching/networking*.

Every group definition file contains a JSON directory with the following
fields.  All fields are optional, but unless you specify at least one user
definition (`op`, `presenter`, or `other`), nobody will be able to join
the group.

 - `op`, `presenter`, `other`: each of these is an array of user
   definitions (see below) and specifies the users allowed to connect
   respectively with operator privileges, with presenter privileges, and
   as passive listeners;
 - `public`: if true, then the group is visible on the landing page;
 - `description`: a human-readable description of the group; this is
   displayed on the landing page for public groups;
 - `max-clients`: the maximum number of clients that may join the group at
   a time;
 - `max-history-age`: the time, in seconds, during which chat history is
   kept (default 14400, i.e. 4 hours);
 - `allow-recording`: if true, then recording is allowed in this group;
 - `allow-anonymous`: if true, then users may connect with an empty username;
 - `allow-subgroups`: if true, then subgroups of the form `group/subgroup`
   are automatically created when first accessed;
 - `redirect`: if set, then attempts to join the group will be redirected
   to the given URL; most other fields are ignored in this case;
 - `codecs`: this is a list of codecs allowed in this group.  The default
   is `["vp8", "opus"]`.
   
Supported video codecs include:

 - `"vp8"` (compatible with all supported browsers);
 - `"vp9"` (better video quality than `"vp8"`, but incompatible with
   older versions of Mac OS);
 - `"h264"` (incompatible with Debian, Ubuntu, and some Android devices,
   recording is not supported).

Supported audio codecs include `"opus"`, `"g722"`, `"pcmu"` and `"pcma"`.
There is no good reason to use anything except Opus.
   
A user definition is a dictionary with the following fields:

 - `username`: the username of the user; if omitted, any username is
   allowed;
 - `password`: if omitted, then no password is required.  Otherwise, this
   can either be a string, specifying a plain text password, or
   a dictionary generated by the `galene-password-generator` utility.
   
For example,

    {"username": "jch", "password": "1234"}
    
specifies user *jch* with password *1234*, while

    {"password": "1234"}
    
specifies that any (non-empty) username will do, and

    {}
    
allows any (non-empty) username with any password.

If you don't wish to store cleartext passwords on the server, you may
generate hashed password with the `galene-password-generator` utility.  A
user entry with a hashed password looks like this:

    {
        "username": "jch",
        "password": {
            "type": "pbkdf2",
            "hash": "sha-256",
            "key": "f591c35604e6aef572851d9c3543c812566b032b6dc083c81edd15cc24449913",
            "salt": "92bff2ace56fe38f",
            "iterations": 4096
        }
    }

--- Juliusz Chroboczek <https://www.irif.fr/~jch/>
