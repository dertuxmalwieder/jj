# jj

A file-based IRC client.

## Index

* [Concepts](#concepts)
* [Dependencies](#dependencies)
* [Installation](#installation)
* [Usage](#usage)
* [Environment Variables](#environment-variables)
* [Directory Structure](#directory-structure)
* [Log Format](#log-format)
* [Input Commands](#input-commands)
* [Hooks](#hooks)
* [Examples](#examples)

### Concepts

jj is an evolution of the [ii(1)][ii homepage] IRC client and it sucks less. It is a small suite of programs consisting of the following three (interchangeable) components:
1. `jjd`
The daemon. It does the bare minimum like connecting to the IRC server and reading user input from a named pipe (fifo). It spawns a child and sends all user and IRC messages to it. Written in C.
2. `jjc`
The client or child if you will. Gets spawned as a child of `jjd` and handles the more typical IRC client things. Written in awk.
3. `jjp`
Pretty prints log files or stdin. Written in awk.

In UNIX, everything is a file. jj tries to follow that mantra by storing the IRC output in various log files, reading user input via a named pipe and handling events in external programs/scripts.

Having log files means you can use your systems text mangling utilities on them (think `grep(1)`). More conventional clients usually have the ability to save log files too but they also tend to implement their own search on top of it, duplicating functionality you already get for free. Log files being the IRC client also means you automatically have a separation between the backend and front-end. You can easily restart the front-end without affecting the backend. Building a front-end is as easy as `tail -f` and some `tmux(1)` usage. You can also write a complex parser in your favorite languange, which colors messages, right aligns nicknames and more. Want to filter out join messages? Use `grep(1)` in the pipeline. Scrollback and search? `tmux(1)`!

Reading input via a named pipe allows you to also script the input side of the front-end. For instance, want to use your editor to write and send IRC messages? No problem!

Common tasks, like auto-joining channels after connecting to an IRC server and various other events, is not handled by jj itself, but instead is delegated to external programs/scripts. That gives the user a lot of freedom and power. To stay with the channel auto-join example: After successfully connecting, `jjc(1)` runs `irc_on_connect` if it's in `PATH`. That program can be written in your favorite languange. It could check the current host and depending on server, join different channels, auth with services etc. To give another example, the `irc_on_highlight` script could be used to send desktop/push notifications. You can get more details about this feature in the [Hooks](#hooks) section.

### Dependencies

Nothing but a C compiler and `awk(1)` with support for time functions.

### Installation

As root user:
```shell
make && make install
```
or as unpriviliged user:
```shell
make && PREFIX=~/.local make install
```
Make sure `~/.local/bin` is in your `PATH`.

### Usage

None of the programs have any options and instead are controlled entirely by environment variables. If you want to use different settings like nickname, host etc, it makes sense to put those variables into your `.profile`, so they don't have to be manually typed for every jj invocation. For a list of variables, consult the [Environment Variables](#environment-variables) section.

To connect to an IRC server, you only have to run:
```shell
jjd
```
or
```shell
JJ_HOST=irc.rizon.net jjd
```
By default it connects to irc.freenode.org using your current user as nickname and creates a directory with host as the name, in the current working directory. Located in that directory will be the various log files and the named pipe for input. For more information, see [Directory Structure](#directory-structure) and [Input Commands](#input-commands).

To save us some typing, lets change into the directory of the new connection:
```shell
cd irc.freenode.org
```

To display the IRC output, the most basic would be to simply run:
```shell
cat server.log
```
`cat` could also be replaced with `jjp` for prettier output.

To join a channel and say something in it, run:
```shell
echo "join #foobar" >in
```
And then:
```shell
echo "msg #foobar Hello, World!" >in
```

You can then read the output of that channel from `channels/#foobar.log`.

See [Input Commands](#input-commands) for a list of input commands and [Examples](#examples) more advanced usage.

### Environment Variables
|Name|Description|Default|
|-|-|-|
|`IRC_CLIENT`|The program spawned as child of `jjd(1)` which handles all the messages.|`jjc`|
|`IRC_DIR`|Where to store the per-host directories.|`.` (Current directory)|
|`IRC_HOST`|The IRC host to connect to.|`irc.freenode.org`|
|`IRC_NICK`|The Nickname to use.|`$USER`|
|`IRC_PASSWORD`|The server password/|unset|
|`IRC_PORT`|Connect using this port.|`6667`|
|`IRC_REALNAME`|The realname to use for the initial authentication. Can also be seen in `whois`.|`$USER`|
|`IRC_USER`|The username to use for the initial authentication.|`$USER`|

When [Hooks](#hooks) are being run, in addition to inheriting the above variables, the following environment variables are also available to the called program:

|Name|Description|
|-|-|
|`IRC_ME`|Your nickname.|
|`IRC_NETWORK`|The networks official name as supplied by the server. Like "QuakeNet".|
|`IRC_TEXT`|The text of a message. If applicable, wise otherwise. If the event was a kick, it would be the kick reason.|
|`IRC_WHERE`|In which channel the event happened. If applicable, empty otherwise.|
|`IRC_WHO`|Who triggered this hook. If applicable, emtpy otherwise.|
|`IRC_CASEMAPPING`|The servers casemapping. For plain ascii, its value would be "A-Z a-z". It can be split on space and then used with `tr(1)` to properly casefold.|
|`IRC_AWAY`|1 when you are marked away, 0 otherwise.|

## Directory structure

```
irc.freenode.org/
├── channels/
│   ├── #channel.log
│   ├── freenode-connect.log
│   └── nickserv.log
├── in
├── motd
└── server.log
```
The *server.log* is where all non channel specific messages go. Instead of spamming *server.log* with the servers message of the day, it is instead written to the file called *motd*. The file named *in* is a named pipe used for sending messages to the IRC server. The *channels* directory contains log files of channels and private messages.

### Input Commands

These are the input commands supported by `jjc` All commands are case-insensitive. "target" is a channel or a nickname. Parameters marked with an asterisk are optional.

|Command|Parameter 1|Parameter 2|Parameter 3|Description|
|-|-|-|-|-|
|`action`|target|message|n/a|Send an action message to a user or channel.|
|`me`|target|message|n/a|An alias for action.|
|`away`|away text*|n/a|n/a|Mark yourself as away. Without parameters it unsets away.|
|`invite`|nickname|channel|n/a|Invite a user to a channel.|
|`join`|channel1[,channel2]...|\*password1[,password2]...|n/a|Join channels.|
|`kick`|channel|nickname|\*reason|Kick a user out of a channel.|
|`list`|n/a|n/a|n/a|Print a list of the currently open channels (including private message channels).|
|`ls`|n/a|n/a|n/a|An alias for list.|
|`message`|target|\*message|n/a|Send a message to a channel or user. When messaging a user without message text, a private message channel will still be created.|
|`message`|target|\*message|n/a|An alias for message.|
|`mode`|target|\*mode|mode parameter|Set various user or channel modes.|
|`names`|channel|n/a|n/a|Request the names listing of a channel.|
|`nick`|nickname|n/a|n/a|Change your nickname.|
|`notice`|target|\*message|n/a|Send a notice to a channel or user. The same rules as with message apply.|
|`part`|target|\*reason|n/a|Leave a channel or close a private message channel.|
|`quit`|\*reason|n/a|n/a|Disconnect from the server and quit jj.|
|`topic`|channel|n/a|n/a|Request the topic of a channel.|
|`topicset`|channel|\*topic|n/a|Set or remove the topic of a channel.|
|`whois`|nickname|n/a|n/a|Request the whois information of a user.|

Additionally, the `raw` command can be used to send unsupported IRC commands without `jjc` interpreting them. Using it to message a channel or user, results in the message not being locally printed (to the log file). It could be used to auth with services to hide passwords.

### Log Format

The general log format is:
```
1579093317 <user123>n! Hello, World!
```
The 1579093317 is the seconds since epoch (UTC). It's easy to convert it to the current timezone and readable format and is what `jjc(1)` already does for you. The second part is the nickname of the author of the message with a suffix indicating message types. The rest is the actual message body.

`jjc(1)` uses the following message type indicators:

* `*` - This is you. You are the author of this message.
* `!` - Important information or your nick is mentioned in this message.
* `n` - This message is a notice.
* `:<chars>:` - A message send only to users with a certain status (in that channel).
* `c` - A CTCP message.
* `a` - an ACTION message.

**Note:** For messages without an author (server messages), a single dash is used as nickname.

### Hooks

Certain events trigger the execution of external programs. Those programs have to be executable and in `PATH` and they are run with an altered environment (See: [Environment Variables](#environment-variables)).

The following programs are searched for a executed:

|Name|Trigger|
|-|-|
|`irc_on_add_channel`|A channel was joined or a new private message window was created.|
|`irc_on_connect`|Succesfully connecting to the server.|
|`irc_on_ctcp`|Receiving a CTCP message.|
|`irc_on_highlight`|Own nick was mentioned in a message.|
|`irc_on_invite`||
|`irc_on_join`||
|`irc_on_kick`||
|`irc_on_part`||

**Note:** When a private message channel is created and the message also contains our nick, instead of triggering `irc_on_add_channel` *and* `irc_on_highlight`, only the former is triggered.

### Examples

Coming soon.

[ii homepage]: https://tools.suckless.org/ii/
