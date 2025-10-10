# Comments from ineiti (Linus Gasser)

I'm a hobby cryptographer involved as a
software engineer in developing similar protocols like
[CoSi](https://arxiv.org/pdf/1602.06997) and
[Calypso](https://eprint.iacr.org/2018/209.pdf).
Currently I'm working on a research grant on evaluating and proposing
advanced cryptographic schemes for electronic identities:
[Secure and Privacy-Preserving Credentials for E-ID](https://github.com/eid-privacy)
This means that I have a pretty good grasp of elementar algebra and saw
more than one way for a protocol to fail.
This also means that I often follow a dead end and get distracted
too often by pretty things...
I hope these suggestions help, and I'm sorry I didn't read the
papers and RFCs...

Also, I wrote this between commits 19270381332689124e406069cf197d90ee16fed5
and 3a8cdae9a4598f724c5a79667ea4b3c24d0a34ec,
so some things might be moot now.

Without other indication, I base myself on the definitions in
[cocktail-dkg.md](./cocktail-dkg.md).

# DKG

This is probably a redundant explanation from the paper, but just to
make sure I get the basics right, this is what I base myself on:

## Input and Output

From a high level, I suppose that every participant has the following
inputs available to them, which have been received over a trusted
channel:

- $n$ public keys $P_1$ .. $P_n$
- the protocol $time$ as milliseconds since the Unix Epoch
- a threshold $t$ who can sign a message
- a minimal number $l$ of live participants needed to finish the protocol
with $t <= l <= n$

The goal of the protocol is to produce the following information:

- the $context = H( "COCKTAIL-DKG-V0.1" | P_1 | ... | P_n | time | t | l )$
- the group public key $Y$
- $n$ secret keys $x_i$, one for each participant $i$
- a common transcript $T = (context || Y || sig_{P_1}, \cdots, sig_{P_n})$

## Protocol Abort and Blame Assignment

If any of the participants detects a cryptographic error in
any of the messages related to a $context$, they must abort
the protocol. They must not accept any other message for this
$context$, including signing requests.
In case of an error from the coordinator, they must send the
$BlameProof$ to all other participants.
In case of an error from another participant, they must call
$CoBlame(context, BlameProof)$.

To make sure that blame assignment works in a reliable manner
without participants or the coordinator being able to blame
an honest entity, care must be taken that all messages are
signed and reliably assignable.
Compared to Cocktail-DKG this means that there are some more
signatures to avoid false blames.

A valid $BlameProof$ from participant $i$ looks like the following:

```math
Blame = (i || context || FaultyMessage || AdditionalInfo)
BlameProof = (Blame || Sig_{P_i}(Blame))
```

To verify a $BlameProof$, only the public keys $P_1, \cdots, P_n$
must be necessary.

The $AdditionalInfo$ is (probably) only used in `Round 2` of the
protocol, in case of a decryption failure, or share verification
failure. In that case, $AdditionalInfo = d_i$ WHICH IS OF COURSE
STUPID, BECAUSE IT'S THIS PARTICIPANTS LONG-TERM PRIVATE KEY!
So either there is another way to securely blame a decryption
failure, or the ephemeral keys must be exchanged beforehand,
and only these must be used for the ECDH.

## DKG Threat Model

For the participants and the coordinator, we consider the following threat model:

- the participants eventually send $msg_{1|i}$, the first step of the protocol
- a number $l$ of _live_ participants follow through with the whole
protocol
- they have limited computing capabilities which does not allow
them to find $x$ given $[ x ] B$
- they will not deviate from the protocol if they can be correctly blamed
(but will happily do so if an honest entity gets blamed)

With regard to the network:

- every message sent over the network eventually gets to the destination
- there is no authentication, encryption, or verification of the message
on the network
- the adversary can read and modify all messages on the network, but will
only do so if it doesn't lead the participants to abort the protocol or to
stall the protocol

## Communication Optimization

The protocol can work in a completely peer-to-peer fashion with
all the particpants communicating directly with each other.
However, this leads to $O(n^2)$ messages to finish the protocol
which can be prohibitive.
For this reason we allow for a coordinator with the following
set of operations.
They are kept generic to allow for different kinds of implementations
like a smart contract or a centralized server:

- $CoID$ is the identity of the coordinator which can be used
to verify proofs of the coordinator.
- $CoVerify(CoID, CoProof, msg)$ -> $()$ or $(Error)$ verifies if the
$msg$ corresponds to the $CoProof$ created by the $CoID$. If the
verification succeeds, it returns empty, else it returns an error.
- $CoStart(P_1, ..., P_n, time, t, l)$ -> $(context)$ or $(Error)$
starts a new DKG. The coordinator returns an error if one or more $P_i$
are invalid curve points, or if $t <= l <= n$ does not hold.
Else the coordinator stores the public
keys and thresholds and returns the $context$.
- $CoMsg1(context, i, msg_{1|i}, sig_{P_i})$ -> $()$ or $(Error)$
allows each participant to store their $msg_{1|i}$. The coordinator
returns an error if the context is not known, $i$ is out of bounds,
the $sig_{P_i}$ does not verify the $msg_{1|i}$, or all
$msg_{1|i}$ are already stored. Else the coordinator stores $msg_{1|i}$ and
$sig_{P_i}$.
- $CoMsg2(context, i)$ -> $(msg_{2|i}, CoProof_msg_{2|i})$ or $(Error)$
allows each participant to retrieve their respective $msg_{2|i}$
for the round 2 of the protocol. The coordinator returns an error if
the context is not known, $i$ is out of bounds, not all participants
sent their $msg_{1|i}$, or a valid $BlameProof$ exists.
Else the coordinator returns $msg_{2|i}$ and the $CoProof_msg_{2|i}$.
- $CoFinalize(context, i, sig_{P_i})$ -> $()$ or $(Error)$
allows each participant to indicate that they successfully finished
the round 2 of the protocol. The coordinator returns an error if the context
is not known, $i$ is out of bounds, the $sig_{P_i}$ does not
verify the message $(context | Y)$, or a valid $BlameProof$ exists.
In case of success it returns empty.
- $CoTranscript(context)$ -> $(T, CoProof_transcript)$ or $(Error)$
allows anybody to get the transcript
of the corresponding context. The coordinator returns an error if
the $context$ is not known, less than $l$ participants finalized
the protocol, or if a valid $BlameProof$ exists.
Else it returns the group public key and at least $l$ valid signatures.
- $CoBlame(context, BlameProof)$ -> $()$ or $(Error)$ aborts the
protocol in case of a valid $BlameProof$. The coordinator returns
an error if the $context$ is not known, or if the $BlameProof$ is
invalid.

### Blockchain

In case of a blockchain, $CoID$ represents the blockchain identity, and
$CoVerify$ must be able to verify that the $CoProof$ is actually
stored somewhere on the blockchain.
The $CoProof$ must point to the transaction and the block where the proof
can be found.
A smart contract can implement the different messages shown here, with the
caveat that the cryptographic operations might be very expensive,
depending on the type of smart contracts available.

### Central Server

For a central server, $CoID$ is the public key of the server,
$CoProof$ is a signature on $msg$, and $CoVerify$ is a signature verification
which returns either an error or an empty result if the verification
is successful.

# Comments and questions

- The document should use section-# and subsection-#, so it would be easier
to reference comments in a document like this :)
- In the $msg2$ I don't understand the $C_{agg}$. I'm used to see "Aggregated"
in elliptic curves as the sum of several elliptic curve points, but here it seems
to be a list of $t-1$ points. However, I don't see how the participants
in round 2 can then use the $C_{j,k}$ if they only have the sum of the points.
So shouldn't the $msg_2$ have all $C_j$ in it?
- Also for the $msg2$, which I changed to $msg_{2|i}$: In the `Protocol Messages`, you
write that the coordinator returns the $msg_{2|i}$ to participant $i$. However, as
described above, this message is not sufficient for participant $i$ to finish
round 2 of the protocol. In the `Round 2` you state that the participants receive
all $msg_{1|i}$ from all other participants, which is too much, as most of the
encrypted shares will not be able to be decrypted. I propose to redefine
$msg_{2|i}$ as follows:

```math
msg_{2|i} = (C_1 || \cdots || C_n) || (PoP_1 || \cdots || Pop_n) || E_1 || \cdots || E_n || c_{1,i} || \cdots || c_{n,i}
```

with all indexes $1 \cdots n$ excluding $i$. And then `Round 2` can state that
each participant fetches $msg_{2|i}$ from the coordinator.

- I would put a visible link to the actual signing protocol!
