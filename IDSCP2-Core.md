The heart of the IDSCP2 protocol is an event-based finite state machine that
consists of several states and transitions, which are triggered by events.

In the following, you can see a highly simplified version of the IDSCP2 state machine:

![](images/FSM.png?raw=true)

The goal of the IDSCP2 handshake is to reach the STATE_ESTABLISHED, which allows
the IDSCP2 communication with the remote peer. To achieve this state, first the handshake
has to be started by the upper layer, valid DATs have to be exchanged and verified, and finally
the RAT drivers have to verify each other's trusted state.

During the handshake, as well as after the protocol is in STATE_ESTABLISHED,
re-attestations and fresh DAT requests can be triggered to keep the communication
trusted over time. This is what makes the state machine so complex, since we have
to handle RAT and DAT timeouts, Upper layer re-attestation requests, as well as
re-attestation- and fresh-DAT-request from the remote peer all the time in every single state.

#### FSM states
- **STATE_CLOSED**, which should be divided into the final state, **STATE_CLOSED_LOCKED**,
  and into the init state, **STATE_CLOSED_UNLOCKED**.

- **STATE_WAIT_FOR_HELLO**, which waits for the IDSCP_HELLO of the remote peer

- **STATE_WAIT_FOR_RAT**, which waits for the result of the RAT_PROVER and RAT_VERIFIER drivers

- **STATE_WAIT_FOR_RAT_PROVER**, which waits for the result of the RAT_PROVER driver

- **STATE_WAIT_FOR_RAT_VERIFIER**, which waits for the result of the RAT_VERIFIER driver

- **STATE_WAIT_FOR_DAT_AND_RAT**, which waits for a fresh DAT of the remote peer, as well as for the RAT_PROVER driver.
  After a valid DAT has been received, the RAT_VERIFIER will be started for re-attestation.

- **STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER**, which waits for a fresh DAT of the remote peer. After
  a valid DAT has been received, the RAT_VERIFIER will be started for re-attestation.

- **STATE_WAIT_FOR_ACK**, which waits for the ACK of the previous message.

- **STATE_ESTABLISHED**, which allows IDSCP2-secure communication with the remote peer.

#### FSM Events
In each of the above mentioned states, each possible FSM event should be handled.
The FSM events are structured into the following categories:

- 4 UPPER_EVENTS: Events from the User / Upper layer (e.g. ReRat, SendData)

- 6 RAT_EVENTS: Events from the RatVerifier- and RatProverDriver (RAT_OK, RAT_FAILURE, RAT_MSG)

- 10 SECURE_CHANNEL_EVENTS: Events from the underlying secure channel (Error, IDSCP2 messages from the remote peer)

- 4 TIMEOUT_EVENTS: Events triggered by Timer Routines (Handshake Timeout, RAT Driver Timeouts, DAT Timeout, Ack Timeout)

In total, there are 24 theoretically possible events per state, which makes in sum 240 transitions
(10 states times 24 possible events). However, there is in general a huge number of events per state,
that can be ignored without hesitation (e.g. almost all events in the final STATE_CLOSED_LOCKED).
In this way, the programming effort can be reduced by about half. However, great care should be taken, as ignoring
important events in specific states can lead to security vulnerabilities.

In detail, the FSM includes the following 24 events:
- **UPPER_START_HANDSHAKE**: Start the IDSCP2 handshake for a fresh Idscp2Connection

- **UPPER_CLOSE**: Close the Idscp2Connection and lock the FSM forever

- **UPPER_SEND_DATA**: Send IDSCP_DATA (application data) to the remote peer

- **UPPER_RE_RAT**: Repeat the RAT verification of the remote peer

- **RAT_VERIFIER_OK**: RAT verification of the remote peer has succeed

- **RAT_VERIFIER_FAILED**: RAT verification of the remote peer has failed

- **RAT_VERIFIER_MSG**: RAT verifier driver has new data for the counterpart remote RAT prover driver

- **RAT_PROVER_OK**: RAT prover has succeed

- **RAT_PROVER_FAILED**: RAT prover has failed

- **RAT_PROVER_MSG**: RAT prover driver has new data for the counterpart remote RAT verifier driver

- **SC_ERROR**: Secure channel failed (e.g. socket IO error)

- **SC_IDSCP_HELLO**: Secure channel received IDSCP_HELLO from remote peer

- **SC_IDSCP_CLOSE**: Secure channel received IDSCP_CLOSE from remote peer

- **SC_IDSCP_DAT**: Secure channel received IDSCP_DAT from remote peer

- **SC_IDSCP_DAT_EXPIRED**: Secure channel received IDSCP_DAT_EXPIRED from remote peer

- **SC_IDSCP_RAT_PROVER**: Secure channel received IDSCP_RAT_PROVER from remote peer

- **SC_IDSCP_RAT_VERIFIER**: Secure channel received IDSCP_RAT_VERIFIER from remote peer

- **SC_IDSCP_RE_RAT**: Secure channel received IDSCP_RE_RAT from remote peer

- **SC_IDSCP_DATA**: Secure channel received IDSCP_DATA from remote peer

- **SC_IDSCP_ACK**: Secure channel received IDSCP_ACK from remote peer

- **HANDSHAKE_TIMEOUT**: A handshake timeout occurred. Lock the FSM forever

- **DAT_TIMEOUT**: A DAT timeout occurred, request a new DAT from remote peer

- **RAT_TIMEOUT**: A RAT timeout occurred, request re-attestation

- **ACK_TIMEOUT**: A ACK timeout occurred, send IDSCP_DATA again

#### Timeouts
The following timeouts exists:

* **RAT Timeout:** This timeout occurs when the trust interval has been expired, and a new re-attestation of
  the remote peer should be requests.

* **DAT Timeout:** This timeout occurs when the DAT of the remote peer has been expired

* **Handshake Timeout:** This timeout occurs when the handshake activity takes to long. It is suggested
  to provide own handshake timers per RAT driver, such that you can restrict the time a RAT
  driver has to prove its trust.

* **Ack Timeout:** This timeout occurs when no IDSCP_ACK has been received for the last sent IDSCP_DATA
  message. It should trigger a repetition of sending the message.

### Detailed FSM state description
In the following, we will discuss for each state what events are important and what transitions
the events will trigger. If any unexpected error occurs, the FSM should go into STATE_CLOSED_LOCKED.

#### STATE_CLOSED_LOCKED
This state is actually the final state of the FSM without any outgoing
transition. If this state has been entered once, it can never ever reach another state again.
Therefore, you have to **ignore every event**.
You should enter this state from any other state whenever an error occurred, or
the connection lose its trust due to an invalid DAT, or a failed re-attestation.

#### STATE_CLOSED_UNLOCKED
This state is actually the init state of the FSM. The only event that should be
recognized in this state is the **UPPER_START_HANDSHAKE** event, which should send an IDSCP_HELLO
to the remote peer via the secure channel and should then go into the state **STATE_WAIT_FOR_HELLO**.

#### STATE_WAIT_FOR_HELLO
This state waits for the remote peer's IDSCP_HELLO, which contains the DAT and the
RAT suites of the remote peer. When the SC_IDSCP_HELLO has been received, you have to verify the
remote DAT via the DAPS driver, calculate the RAT mechanisms and start the RAT drivers.
A peer always decides which RAT mechanism should be used to verify the remote peer. Thus, your outer loop
in your matching algorithm always has to loop through the RAT_VERIFIERS, while the inner loop
goes through the counterpart RAT_PROVERS.
In this state the handshake timer should be active.

You should handle only the following events:
- HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- SC_ERROR: go to STATE_CLOSED_LOCKED

- SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

- SC_IDSCP_HELLO: If the DAT is valid, start the DAT timeout, calculate the RAT mechanisms,
  start the RAT drivers, start the driver handshake timeouts and go to STATE_WAIT_FOR_RAT. Otherwise,
  send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

#### STATE_WAIT_FOR_RAT
This state waits for the results of the RAT_PROVER, as well as the RAT_VERIFIER driver.
The driver handshake timer, as well as the DAT timer should be active.

You should handle only the following events:
- HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- DAT_TIMEOUT: send IDSCP_DAT_EXPIRED and go to STATE_WAIT_FOR_DAT_AND_RAT

- UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- RAT_PROVER_OK: cancel the RAT_PROVER handshake timer and go to STATE_WAIT_FOR_RAT_VERIFIER

- RAT_PROVER_MSG: send IDSCP_RAT_PROVER and stay in STATE_WAIT_FOR_RAT

- RAT_PROVER_FAILED: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- RAT_VERIFIER_OK: cancel the RAT_VERIFIER handshake timer and go to STATE_WAIT_FOR_RAT_PROVER

- RAT_VERIFIER_MSG: send IDSCP_RAT_VERIFIER and stay in STATE_WAIT_FOR_RAT

- RAT_VERIFIER_FAILED: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- SC_ERROR: go to STATE_CLOSED_LOCKED

- SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

- SC_IDSCP_DAT_EXPIRED: send a fresh IDSCP_DAT, restart RAT_PROVER and stay in STATE_WAIT_FOR_RAT

- SC_IDSCP_RAT_PROVER: delegate to local RAT_VERIFIER driver and stay in STATE_WAIT_FOR_RAT

- SC_IDSCP_RAT_VERIFIER: delegate to local RAT_PROVER driver and stay in STATE_WAIT_FOR_RAT

- SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag and alternate the next_send_alternating_bit, otherwise ignore. Stay in STATE_WAIT_FOR_RAT

#### STATE_WAIT_FOR_RAT_VERIFIER
This state waits for the result of the RAT_VERIFIER driver. The RAT_VERIFIER handshake timer, as well as
the DAT timer should be active.

You should handle only the following events:
- HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- DAT_TIMEOUT: send IDSCP_DAT_EXPIRED and go to STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER

- UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- RAT_VERIFIER_OK: cancel the RAT_VERIFIER handshake timer and go to STATE_ESTABLISHED or STATE_WAIT_FOR_ACK
  depending on if the ack flag is set or not.

- RAT_VERIFIER_MSG: send IDSCP_RAT_VERIFIER and stay in STATE_WAIT_FOR_RAT_VERIFIER

- RAT_VERIFIER_FAILED: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- SC_ERROR: go to STATE_CLOSED_LOCKED

- SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

- SC_IDSCP_DAT_EXPIRED: send a fresh IDSCP_DAT, restart RAT_PROVER and stay in STATE_WAIT_FOR_RAT

- SC_IDSCP_RAT_PROVER: delegate to local RAT_VERIFIER driver and stay in STATE_WAIT_FOR_RAT_VERIFIER

- SC_IDSCP_RE_RAT: restart the RAT_PROVER driver and go to STATE_WAIT_FOR_RAT

- SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag and alternate the next_send_alternating_bit, otherwise ignore. Stay in STATE_WAIT_FOR_RAT_VERIFIER

#### STATE_WAIT_FOR_RAT_PROVER
This state waits for the result of the RAT_PROVER driver. The RAT_PROVER handshake timer, as well as
the DAT timer, and the RAT timer should be active. In this state, the remote peer has already been verified.

You should handle only the following events:
- HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- RAT_TIMEOUT: request re-attestation by sending IDSCP_RE_RAT, start the RAT_VERIFIER and its handshake timeout
  and go to STATE_WAIT_FOR_RAT

- DAT_TIMEOUT: send IDSCP_DAT_EXPIRED and go to STATE_WAIT_FOR_DAT_AND_RAT

- UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- UPPER_RE_RAT: request re-attestation by sending IDSCP_RE_RAT, start the RAT_VERIFIER and its handshake timeout
  and go to STATE_WAIT_FOR_RAT

- RAT_PROVER_OK: cancel the RAT_PROVER handshake timer and go to STATE_ESTABLISHED or STATE_WAIT_FOR_ACK
  depending on if the ack flag is set or not.

- RAT_PROVER_MSG: send IDSCP_RAT_PROVER and stay in STATE_WAIT_FOR_RAT_PROVER

- RAT_PROVER_FAILED: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

- SC_ERROR: go to STATE_CLOSED_LOCKED

- SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

- SC_IDSCP_RE_RAT: restart the RAT_PROVER driver and stay in STATE_WAIT_FOR_RAT_PROVER

- SC_IDSCP_DAT_EXPIRED: send a fresh IDSCP_DAT, restart RAT_PROVER and stay in STATE_WAIT_FOR_RAT_PROVER

- SC_IDSCP_RAT_VERIFIER: delegate to local RAT_PROVER driver and stay in STATE_WAIT_FOR_RAT_PROVER

- SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag and alternate the next_send_alternating_bit, otherwise ignore. Stay in STATE_WAIT_FOR_RAT_PROVER


#### STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER
In this state the local peer first has to wait for a fresh valid DAT from the remote peer. Only the
handshake timer should be active. No RAT driver should be active.

You should handle only the following events:
* HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* SC_ERROR: go to STATE_CLOSED_LOCKED

* SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

* SC_IDSCP_DAT_EXPIRED: send a fresh IDSCP_DAT, start the RAT_PROVER driver and go to STATE_WAIT_FOR_DAT_AND_RAT

* SC_IDSCP_DAT: if the DAT is valid, start the DAT timeout, start the RAT_VERIFIER driver and go to STATE_WAIT_FOR_RAT_VERIFIER

* SC_IDSCP_RE_RAT: start the RAT_PROVER driver and go to STATE_WAIT_FOR_DAT_AND_RAT

* SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag and alternate the next_send_alternating_bit, otherwise ignore. Stay in
  STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER

#### STATE_WAIT_FOR_DAT_AND_RAT
This state is similar to the STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER, but the RAT_PROVER
driver is active. Only the handshake timer should be active.

You should handle only the following events:
* HANDSHAKE_TIMEOUT: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* RAT_PROVER_OK: cancel the RAT_PROVER handshake timer and go to STATE_WAIT_FOR_DAT_AND_RAT

* RAT_PROVER_MSG: send IDSCP_RAT_PROVER and stay in STATE_WAIT_FOR_DAT_AND_RAT

* RAT_PROVER_FAILED: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* SC_ERROR: go to STATE_CLOSED_LOCKED

* SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

* SC_IDSCP_DAT_EXPIRED: send a fresh IDSCP_DAT, restart the IDSCP_RAT_PROVER and stay in
  STATE_WAIT_FOR_DAT_AND_RAT

* SC_IDSCP_DAT: if the DAT is valid, start the DAT timeout, start the RAT_VERIFIER driver and go to
  STATE_WAIT_FOR_RAT

* SC_IDSCP_RAT_VERIFIER: delegate to local RAT_PROVER driver and stay in STATE_WAIT_FOR_DAT_AND_RAT

* SC_IDSCP_RE_RAT: restart the RAT_PROVER driver and stay in STATE_WAIT_FOR_DAT_AND_RAT

* SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag and alternate the next_send_alternating_bit, otherwise ignore. Stay in
  STATE_WAIT_FOR_DAT_AND_RAT

#### STATE_ESTABLISHED
This state is the goal state, which allows secure and trusted communication between
two peers. The DAT timeout and the RAT timeout should be active in this state.

You should handle only the following events:
* UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* UPPER_RE_RAT: send IDSCP_RE_RAT, start RAT_VERIFIER driver and RAT_VERIFIER handshake timer and
  go to STATE_WAIT_FOR_RAT_VERIFIER

* UPPER_SEND_DATA: send IDSCP_DATA with next_send_alternating bit, message, set ACK flag, start ACK timer
  and go to STATE_WAIT_FOR_ACK

* RAT_TIMEOUT: send IDSCP_RE_RAT, start RAT_VERIFIER driver and RAT_VERIFIER handshake timer and
  go to STATE_WAIT_FOR_RAT_VERIFIER

* DAT_TIMEOUT: cancel RAT timer, send IDSCP_DAT_EXPIRED and go to STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER

* SC_ERROR: go to STATE_CLOSED_LOCKED

* SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

* SC_DAT_EXPIRED: send a fresh IDSCP_DAT, restart the local RAT_PROVER and go to
  STATE_WAIT_FOR_RAT_PROVER

* SC_RE_RAT: start the RAT_PROVER driver and go to STATE_WAIT_FOR_RAT_PROVER

* SC_IDSCP_DATA: If the received alternating bit within the IDSCP_DATA is not equal to the
  expected_alternating bit, ignore the message. Otherwise, send an IDSCP_ACK containing the expected_alternating
  bit to the remote peer, then alternate the expected_alternating bit and pass the message to the upper layer.

#### STATE_WAIT_FOR_ACK
This state is a kind of established state, however before new messages can be sent, we
have to wait for the expected IDSCP_ACK that belongs to our previous sent message. This is required to
guarantee that packages cannot be lost during a re-attestation of the trusted state.
The IDSCP_ACK follows the AlternatingBit protocol, which is explained below. An ACK timer will be active
to trigger repetition of sending data when no ACK has been received within a specified interval.
Further, the RAT timer and the DAT timer should be active.

You should handle only the following events:
* UPPER_CLOSE: send IDSCP_CLOSE and go to STATE_CLOSED_LOCKED

* UPPER_RE_RAT: cancel ACK timer, send IDSCP_RE_RAT, start RAT_VERIFIER driver and RAT_VERIFIER handshake timer and
  go to STATE_WAIT_FOR_RAT_VERIFIER

* RAT_TIMEOUT: cancel ACK timer, send IDSCP_RE_RAT, start RAT_VERIFIER driver and RAT_VERIFIER handshake timer and
  go to STATE_WAIT_FOR_RAT_VERIFIER

* DAT_TIMEOUT: cancel RAT timer and ACK timer, send IDSCP_DAT_EXPIRED and go to STATE_WAIT_FOR_DAT_AND_RAT_VERIFIER

* ACK_TIMEOUT: repeat sending the cached IDSCP_DATA, restart the ACK timer and stay in STATE_WAIT_FOR_ACK

* SC_ERROR: go to STATE_CLOSED_LOCKED

* SC_IDSCP_CLOSE: go to STATE_CLOSED_LOCKED

* SC_DAT_EXPIRED: cancel ACK timer, send fresh IDSCP_DAT, restart the RAT_PROVER driver and
  go to STATE_WAIT_FOR_RAT_PROVER

* SC_RE_RAT: cancel ACK timer, start the RAT_PROVER driver and go to STATE_WAIT_FOR_RAT_PROVER

* SC_IDSCP_DATA: If the received alternating bit within the IDSCP_DATA is not equal to the
  expected_alternating bit, ignore the message. Otherwise, send an IDSCP_ACK containing the expected_alternating
  bit to the remote peer, then alternate the expected_alternating bit and pass the message to the upper layer.

* SC_IDSCP_ACK: if this ACK belongs to the expected message regarding the alternating bit protocol,
  clear the ack flag, alternate the next_send_alternating_bit and go to STATE_ESTABLISHED, otherwise ignore
  it and stay in STATE_WAIT_FOR_ACK

### AlternatingBit Protocol
The alternating bit protocol is responsible for guaranteeing that no IDSCP_DATA can be got lost
during a re-attestation procedure. Each peer has one **next_send_alternating_bit** for sending
IDSCP_DATA without loss and one **expected_alternating_bit** for receiving IDSCP_DATA in the right order.

At the beginning, both bits must be initialized with zero. When sending new IDSCP_DATA, the
**next_send_alternating_bit** is placed into the IDSCP_DATA message. It is only then alternated, when
an IDSCP_ACK with this same **next_send_alternating_bit** has been received.

The **expected_alternating_bit** is used for checking if a new received IDSCP_DATA message has correctly
been ordered and no other IDSCP_DATA between has been lost between. When the bit of the received IDSCP_DATA
matches the **expected_alternating_bit** we will send an IDSCP_ACK with this
**expected_alternating_bit** and we will alternate it afterwards.

### RatMechanism Calculation
As already described above, each peer has to decide its local RAT_VERIFIER mechanism, which is
then used by the local RAT_VERIFIER, as well as by the corresponding remote RAT_PROVER to verifier that the
remote peer is in a trusted state.
The local RAT suites are passed by the user to the IDSCP core, while the remote suites are received
via the IDSCP_HELLO, which holds a list of supported and expected RAT mechanisms. These lists are
ordered by the priority, which means that early entries should be preferred.

An implementation could look like the following one:
```kotlin
fun calculateRatMechanism(verifier_suites: Array<String>, prover_suites: Array<String>): String? {
    if (verifier_suites.isNullOrEmpty() || prover_suites.isNullOrEmpty()) {
        return null;
    }
    
    for (v in verifier_suites) {
        for (p in prover_suites) {
            if (p == s) {
                // found a match
                return p
            }
        }
    }
    
    // no match
    return null
}

```
This is then called twice to calculate both, the local RatVerifier and the local RatProver mechanism:
```kotlin
private val localVerifierSuites: Array<String>
private val localProverSuites: Array<String>
private lateinit var proverMechanism: String?
private lateinit var verifierMechanism: String?

...

fun receivedIdscpHello(hello: IdscpHello) {
    
    ...
    
    // remote peer decides
    this.proverMechanism = calculateRatMechanism(hello.verifier_suites, this.localProverSuites)
    // we decide
    this.verifierMechanism = calculateRatMechanism(this.localVerifierSuites, hello.prover_suites)

    ...
 
}
```

### IDSCP message format
The IDSCP messages have been built via Protobuf version 3.
The following format must be respected in all implementations:

```proto
syntax = "proto3";

//IDSCP message frame
message IdscpMessage {
    // One of the following will be filled in.
    oneof message {
        IdscpHello idscpHello = 1;
        IdscpClose idscpClose = 2;
        IdscpDatExpired idscpDatExpired = 3;
        IdscpDat idscpDat = 4;
        IdscpReRat idscpReRat = 5;
        IdscpRatProver idscpRatProver = 6;
        IdscpRatVerifier idscpRatVerifier = 7;
        IdscpData idscpData = 8;
        IdscpAck idscpAck = 9;
    }
}


//IDSCP messages
message IdscpHello {
    int32 version = 1;                      //IDSCP protocol version
    IdscpDat dynamicAttributeToken = 2;     //initial dynamicAttributeToken
    repeated string supportedRatSuite = 3;  //RemoteAttestationCipher prover
    repeated string expectedRatSuite = 4;   //RemoteAttestationCipher verifier
}

message IdscpClose {

    enum CloseCause {
        USER_SHUTDOWN = 0;
        TIMEOUT = 1;
        ERROR = 2;
        NO_VALID_DAT = 3;
        NO_RAT_MECHANISM_MATCH_PROVER = 4;
        NO_RAT_MECHANISM_MATCH_VERIFIER = 5;
        RAT_PROVER_FAILED = 6;
        RAT_VERIFIER_FAILED = 7;
    }

    CloseCause cause_code = 1;
    string cause_msg = 2;
}

message IdscpDatExpired {}

message IdscpDat {
    bytes token = 1;
}

message IdscpReRat {                
    string cause = 1;               
}

message IdscpRatProver {
    bytes data = 1;
}

message IdscpRatVerifier {
    bytes data = 1;
}

message IdscpData {                 
    bytes data = 1;
    bool alternating_bit = 2;
}

message IdscpAck {
    bool alternating_bit = 1;
}
```
