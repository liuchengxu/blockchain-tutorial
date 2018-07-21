[The Bitcoin Backbone Protocol: Analysis and Applications](https://eprint.iacr.org/2014/765.pdf)


# Ouroboros: A Provably Secure Proof-of-Stake Blockchain Protocol

- Proof of Stake
- Proof of physical resources(e.g., proof of work)


First, we provide a model that formalizes the problem of realizing a PoS-based blockchain protocol. The model we introduce is in the spirit of [24], focusing on persistence and liveness,

- *persistence*
- *liveness*


coin-flipping protocol ==> produce the randomness for the leader election process


Also, unique to our approach is the fact that the system ignores round-to-round stake modifications. Instead, a snapshot of the current set of stakeholders is taken in regular
intervals called epochs; in each such interval a secure multiparty computation takes place utilizing the blockchain itself as the broadcast channel.


Specifically, in each epoch a set of randomly selected stakeholders form a committee which is then responsible for executing the coin-flipping protocol. The outcome of the protocol determines the set of next stakeholders to execute the protocol in the next epoch as well as the outcomes of all leader elections for the epoch.


-  a snapshot of the current set of stakeholders is taken in regular intervals called epochs

-  in each such interval a secure multiparty computation takes place utilizing the blockchain itself as the broadcast channel


Our protocol is secure under a number of plausible assumptions:
(1) the network is synchronous in the sense that an upper bound can be determined during which any honest stakeholder is able to communicate with any other stakeholder,
(2) a number of stakeholders drawn from the honest majority is available as needed to participate in each epoch,
(3) the stakeholders do not remain offline for long periods of time,
(4) the adaptivity of corruptions is subject to a small delay that is measured in rounds linear in the security parameter (or alternatively, the players have access to a sender-anonymous broadcast channel).

Sleepy consensus considers a fixed stakeholder distribution (i.e., stake does not evolve over time) and targets a “mixed” corruption setting, where the adversary is allowed to be adaptive as well as perform fail-stop and recover corruptions in addition to Byzantine faults.

Snow White [7] addresses an evolving stakeholder distribution and uses a corruption delay mechanism similar to ours for arguing security. Nevertheless, contrary to our protocol, the Snow White design is susceptible to a “grinding” type of attack that can bias high probability events in favor of the adversary

Algorand

- [Ouroboros: A Provably Secure Proof-of-Stake Blockchain Protocol](https://eprint.iacr.org/2016/889.pdf)
