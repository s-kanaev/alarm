
# Protocol for automobile alarm system

There are only two endpoints in automobile alarm system: dongle/trinket and the alarm system installed in auto.

Lets name dongle/trinket as just dongle and we will call the alarm system installed in automobile as just the alarm.

This protocol provides means to make it harder to create a duplicate of dongle even though some intruder has a snapshot of a single lock or unlock packet frame.
User preferences are available to make obtaining of a duplicate of dongle even harder if a intruder has a snapshot of series of lock/unlock packet frames.

Each and every pair of alarm-dongle is identified with a single ID.

## Packet frame contents

A single packet frame consists of header and payload.
Header is followed by payload without any padding bytes.

### Packet header
A header consists of multiple fields:

| hex length | field name           | designation |
| ---------- | -------------------- | ----------- |
| 04         | prefix               | `pfx`       |
| 01         | protoccol version    | `p_ver`     |
| 04         | modified id          | `*id`       |
| 01         | modified packet type | `*ptype`    |
| 01         | payload length       | `d_len`     |
| 02         | payload crc          | `d_crc`     |
| 01         | header crc           | `h_crc`     |

Prefix is only employed for signalling that there is a correct packet.
Protocol version is self-describing field.
Value which is transmitted in modified id field is calculated with following formula:

```
*id = ID_FUNC(ID_SALT, ID, ITER)
```

In this formula:
 - `ID_FUNC` is some user-defined function;
 - `ID_SALT` is user-defined constant (salt);
 - `ID` is manufacturer-defined identifier of alarm-dongle pair;
 - `ITER` is iteration number, which is described later.

Value which is transmitted in modified packet type is calculated with similar formula:

```
*ptype = PTYPE_FUNC(PTYPE_SALT, PTYPE, ITER)
```

Definitions of names, employed in this formula is similar to previous formula, except for `PTYPE`, which is the original packet type.
`PTYPE` is one of the following:

| PTYPE        | direction       | meaning                                                                       |
| ------------ | --------------- | ----------------------------------------------------------------------------- |
| `PT_CMD`     | dongle => alarm | command to be executed by alarm system                                        |
| `PT_CMD_ACK` | alarm => dongle | acknowledge on command recept and execution status                            |
| `PT_STATE`   | alarm => dongle | current status of alarm system (incl. permiter breach and self-diagnosis info |

Payload length is self-describing field as well as payload CRC and header CRC.

**NOTE**: Header CRC is calculated using all fields of header with header CRC byte equal to 0x00.

### Packet payload

#### `PT_CMD`

Command to be executed by alarm system.
Main contents of payload is:

| hex size | designation | meaning            |
| -------- | ----------- | ------------------ |
| 04       | `cmd_id`    | ID of command      |
| 01       | `cmd_type`  | what to do         |
| xx       | `cmd_ctx`   | context of command |

ID of command is employed for acknowledging its recept and report execution status.
`cmd_id` value is generated at dongle by generator function `NEXT_CMD_ID()`.

#### `PT_CMD_ACK`

Acknowledges that command of the given ID was successfully recept.
Reception of this packet by dongle is what is needed for dongle to know, that the command with given ID is being recept by the alarm system.
Contents of payload is:

| hex size | designation | meaning                                   |
| -------- | ----------- | ----------------------------------------- |
| 04       | `cmd_id`    | ID of recept (and maybe executed) command |
| 01       | `exec_ack`  | execution status                          |


Execution acknowledgement status is either of but not limited to:

| hex value | status      | meaning                                         |
| --------- | ----------- | ----------------------------------------------- |
| 01        | `EA_RCP`    | command is only being recept, execution started |
| 02        | `EA_E_SUCC` | command is executed successfully                |
| 03        | `EA_E_FAIL` | command wasn't executed due to some failure     |

`EA_RCP` is temporal acknowledgement status which only means 'OK, I've got your request and am going to execute it.'
`EA_E_SUCC` and `EA_E_FAIL` are final acknowledgement status which means that command is executed successfully or with fault.

#### `PT_STATE`

This packet contains data on status of security permiter.
The packet is regularly sent.
Time delay between sendings is defined by state of the alarm system.
If there is security breach detected these packets are sent more frequently.

Contents of the packet is a **TODO**.

### Command's lifecycle

State diagram of command at **dongle**:

```
                   ,---------------------------------------.
                   |                                       |
                   |                                      \|/
 ---------      ---*----      -------------------      ----------      ------------      -------------
 | START | ===> | sent | ===> | awaiting recept | ===> | recept | ===> | executed | ===> | DISMISSED |
 ---------      --------      ---------.---------      ----------      ------------      -------------
                   ^                   |                                                       ^
                   |                   |                                                       |
                   `-------------------*-------------------------------------------------------'
```

At dongle we'll call `sent` and `awaiting recept` states an unacknowledged states.
`recept` and `executed` states will be called as acknowledged states.


State diagram of command at **alarm system**:

```
 ---------      ----------      ------------      -------------
 | START | ===> | recept | ===> | executed | ===> | DISMISSED |
 ---------      ----------      ------------      -------------
```


Lifetime cycle of command packet:

```
    ----------                         ----------------
    | dongle |                         | alarm system |
    ----.-----                         -------.--------
        |                                     |
        | (The user pushes a button.          |
        |  Command with ID=cid is created.    |
        |  Command is in state 'START'.)      |
        |                                     |
        | (Command is about to be sent.       |
        |  Transit to state 'sent'.)          |
        |----------- PT_CMD(cid) ------------>|
        |                                     |
        | (Command is being sent.             |
        |  Transit to state 'awaiting recept' |
        |                                     | (The command is being parsed.
        |                                     |  Command is in state 'START'.)
        |                                     |
        |                                     | (Command is accepted.
        |                                     |  Transit to state 'recept'.)
        |                                     | (ITER is incremented.)
        |<----- PT_CMD_ACK(cid, EA_RCP) ------|
        | (Transit to state 'recept'.)        |
        | (ITER is incremented.)              |
        |                                     |
        |                                     | (Command is being executed.
        |                                     |  Transit to state 'executed'.)
        |<----- PT_CMD_ACK(cid, EA_E_*) ------|
        | (Transit to state 'executed'.)      | (Transit to state 'DISMISSED'.)
        | (Display result to user.)           | (Erase command from the queue.)
        | (Transit to state 'DISMISSED'.)     .
        | (Erase command from the queue.)     .
        .                                     .
        .                                     .
        .                                     .
```

Commands generated by user are to be put to the sending queue. The queue isn't shown in the diagram for simplicity.
All the commands, which are in the queue are in `START` state.
Only one command is allowed in any work state (i.e. neither `START` nor `DISMISSED`).
That means that after sending of command with `cid1` neither command will be sent/resent until `cid1` is being acknowledged with any of final acknowledgement status (i.e. `EA_E_*`).
Also, acknowledgments are only accepted for command being currently sent (i.e. `cid1`).

`DISMISSED` commands are to be removed/erased from memory.

At dongle command can transit `sent` -> `recept` in case of `PT_CMD_ACK(cid, EA_E_*)` received prior to `PT_CMD_ACK(cid, EA_RCP)`.

#### Resending of unacknowledged commands

Automatic resending of command is only allowed if it is in `awaiting recept` state for too long.
How long is "too long" here is **TBD**.

### The magic behind `ITER` field

`ITER` is incremented when command with `ID=cid` transits to any of acknowledged states at dongle.
Though, transmit/recept is continued with unincremented version of `ITER` for command with `ID=cid`.
Unincremented value to `ITER` is called `PREV_ITER`.

After command with `ID=cid` is dismissed the alarm system will regularly send `PT_STATE` with `*id` and `*ptype` calculated with `PREV_ITER`.
At the same time incoming packets are validated (i.e. values `*id` and `*ptype` are checked) with current value of `ITER`.

This makes it quite uncomfortable for intruder to create a copy of dongle as value of `*id` and `*ptype` is constantly changing.
Though one may obtain a long sequential series of `PT_CMD+PT_CMD_ACK+PT_STATE` (you'll have to track the vehicle within low range distance for quite a long time for this to happen).
To overcome this case the user may employ a more complex functions `ID_FUNC` and `PTYPE_FUNC`.

