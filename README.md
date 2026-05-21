# eBlocker Environment Simulator

The component `eblocker-icapserver` interacts with the system via
Redis channels (publish/subscribe) and via the `script_wrapper`
program.

Redis channels are used to read and write network packets like ARP
requests and responses via low-level C programs.

The `script_wrapper` program can call scripts in a defined directory
and runs as root, so the `eblocker-icapserver` can start/stop services
and configure the OS.

The purpose of the eBlocker Environment Simulator is to

* simulate the `script_wrapper` and the scripts it can run
* simulate the low-level C programs subscribing to and publishing into
  specific Redis channels
* simulate various failure cases that can occur on a real eBlocker
  system.

The simulator is a program running in the terminal that connects to
Redis and subscribes to specific Redis channels at startup.

The simulator has internal state, for example it has a flag:

* is the eBlocker updating currently?

This state is stored in Redis in the hash `simulator`.

For example, the updating state is stored as

    redis-cli> hset simulator updating true

## Implementation

The simulator is implemented in Go. It uses the
[go-redis](https://github.com/redis/go-redis/) client.

It has two components:

* the main simulator, running continuously
* the simulated `script_wrapper` that is run on demand by the
  `eblocker-icapserver`.

### Communication between simulator components

The `script_wrapper` is called with various numbers of arguments:

* name of the script to run
* optional command line arguments.

It first generates a new integer from the Redis sequence (INCR)
`simulator_script_wrapper`. This integer is the `callID`.

It subscribes to three Redis channels:

* `simulator_script_wrapper:<callID>:stdout` for strings to print to STDOUT,
* `simulator_script_wrapper:<callID>:stderr` for strings to print to STDERR,
* `simulator_script_wrapper:<callID>:return` for the return code of the script.

It then publishes the `callID` and the list of command line arguments
to the channel `simulator_script_wrapper:in` as a JSON array, e.g.

    [23, "myscript", "arg1", "arg2"]

When the wrapper receives the integer return code on the `...:return`
channel, it quits and returns the code.

