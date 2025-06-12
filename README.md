[![Documentation](
    https://img.shields.io/readthedocs/bottom-docs?style=for-the-badge)](
    http://bottom-docs.readthedocs.org/)
[![PyPi](
    https://img.shields.io/pypi/v/bottom?style=for-the-badge)](
    https://pypi.org/project/bottom/)
[![GitHub License](
    https://img.shields.io/github/license/numberoverzero/bottom?style=for-the-badge)](
    https://github.com/numberoverzero/bottom/blob/master/LICENSE)
[![GitHub Issues or Pull Requests](
    https://img.shields.io/github/issues/numberoverzero/bottom?style=for-the-badge)](
    https://github.com/numberoverzero/bottom/issues)

# asyncio-based rfc2812-compliant IRC Client (3.12+)

bottom is a small no-dependency library for running simple or complex IRC clients.

It's easy to get started with built-in support for common commands, and extensible
enough to support any capabilities, including custom encryption, local events,
bridging, replication, and more.

# Installation

```
pip install bottom
```

# Documentation

The user guide and API reference are [available here](http://bottom-docs.readthedocs.io/) including
examples for regex based routing of privmsg, custom encryption, and a full list of
[rfc2812 commands](https://bottom-docs.readthedocs.io/en/latest/user/commands.html) that are supported by default.

# Quick Start

The following example creates a client that will:
* connect, identify itself, wait for MOTD, and join a channel
* respond to `PING` automatically
* respond to any `PRIVMSG` sent directly to it, or in a channel


```py
import asyncio
import bottom

host = 'chat.freenode.net'
port = 6697
ssl = True

NICK = "bottom-bot"
CHANNEL = "#bottom-dev"

bot = bottom.Client(host=host, port=port, ssl=ssl)


@bot.on('CLIENT_CONNECT')
async def connect(**kwargs):
    await bot.send('nick', nick=NICK)
    await bot.send('user', user=NICK,
                realname='https://github.com/numberoverzero/bottom')

    # Don't try to join channels until we're past the MOTD
    await bottom.wait_for(bot, ["RPL_ENDOFMOTD", "ERR_NOMOTD"])

    await bot.send('join', channel=CHANNEL)


@bot.on('PING')
async def keepalive(message: str, **kwargs):
    await bot.send('pong', message=message)


@bot.on('PRIVMSG')
async def message(nick: str, target: str, message: str, **kwargs):
    if nick == NICK:
        return  # bot sent this message, ignore
    if target == NICK:
        target = nick  # direct message, respond directly
    # else: respond in channel
    await bot.send("privmsg", target=target, message=f"echo: {message}")


async def main():
    await bot.connect()
    try:
        # serve until the connection drops...
        await bot.wait("client_disconnect")
        print("\ndisconnected by remote")
    except asyncio.CancelledError:
        # ...or we hit ctrl+c
        await bot.disconnect()
        print("\ndisconnected after ctrl+c")


if __name__ == "__main__":
    asyncio.run(main())
```

# API

The public API is fairly small, and built around sending commands with
`send(cmd, **kw)` (or `send_message(msg)` for raw IRC lines) and processing
events with `@on(event)` and `wait(event)`.

```py
class Client:
    # true when the underlying connection is closed or closing
    is_closing() -> bool:

    # connects to the given host, port, and optionally over ssl.
    async connect() -> None

    # start disconnecting if connected.  safe to call multiple times.
    async disconnect() -> None

    # send a known rfc2812 command, formatting kwargs for you
    async send(command: str, **kwargs) -> None

    # decorate a function (sync or async) to handle an event.
    # these can be rfc2812 events (privmsg, ping, notice) or built-in
    # events (client_connect, client_disconnect) or your own signals
    @on(event: str)(async handler)

    # manually trigger an event to be processed by any registered handlers
    # for example, to simulate receiving a message:
    #     my_client.trigger("privmsg", nick=...)
    # or send a local-only message to another part of your system:
    #     trigger("backup-local", backend="s3", session=...)
    trigger(event: str, **kwargs) -> asyncio.Task

    # wait for an event to be triggered.
    async wait(event: str) -> dict

    # send raw IRC line.  bypasses rfc2812 parsing and validation,
    # so you can support custom IRC messages or extensions, like SASL.
    async send_message(message: str) -> None

    # functions that handle the inbound raw IRC lines.
    # by default, Client includes an rfc2812 handler that triggers
    # events caught by @Client.on
    message_handlers: list[ClientMessageHandler]

# wait for the client to emit one or more events.  when mode is "first"
# this returns the events that finished first (more than one event can be triggered
# in a single loop step) and cancels the rest.  when mode is "all" this waits
# for all events to trigger.
async def wait_for(client, events: list[str], mode: "first"|"all") -> list[dict]

# type hints for message handlers
type NextMessageHandler[T: Client] = Callable[[bytes], T, Coroutine[Any, Any, Any]]
type ClientMessageHandler[T: Client] = Callable[[NextMessageHandler[T], T, bytes], Coroutine[Any, Any, Any]]
```

# Contributors

* [fahhem](https://github.com/fahhem)
* [thebigmunch](https://github.com/thebigmunch)
* [tilal6991](https://github.com/tilal6991)
* [AMorporkian](https://github.com/AMorporkian)
* [nedbat](https://github.com/nedbat)
* [Coinkite Inc](https://github.com/coinkite)
* [Johan Lorenzo](https://github.com/JohanLorenzo)
* [Dominik Miedziński](https://github.com/miedzinski)
* [Yay295](https://github.com/Yay295)
* [Elijah Lazkani](https://github.com/elazkani)
* [hell-of-the-devil](https://github.com/hell-of-the-devil)
