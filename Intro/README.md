# Introduction to TPMs

Trusted Platform Modules (TPMs) are a large and complex topic, made all
the more difficult to explain by the intricate relationships between the
relevant concepts.  This is an attempt at a simple explanation -- much
simpler than reading hundreds of pages of documents, but then too, too
light on detail to be immediately useful.

So what is a TPM?  Well, it's a cryptographic co-processor with special
features to enable "root of trust measurement" (RTM), remote attestation
of system state, unlocking of local resources that are kept encrypted
(e.g., filesystems), and more.  A TPM can do those things, and it can do
it with rich authentication and authorization policies.

> The standards development organization that publishes TPM specifications
> is the [Trusted Computing Group (TCG)](https://trustedcomputinggroup.org).

Typically a TPM is a hardware module, a chip, though there are firmware,
virtual, and simulated TPMs as well, all implemented in software.

To simplify things we'll consider only TPM 2.0.

Other parts of this [tutorial](README.md) may cover specific concepts in
much more detail.

# Goals

The goal of this introductory material is to help readers new to TPMs to
understand them well enough to approach the subjects of:

 - [attestation](/Attestation/README.md)
 - [secure boot](/Boot-with-TPM/README.md)

and to think about the sorts of things that one can do with TPMs in
general, which include:

 - device on-boarding
 - ascertaining the state of a device (e.g., has it executed only
   trusted code)
 - unlocking of devices using TPM-based authentication and authorization
   policies (e.g., unlocking a laptop on boot multiple factors such as
   biometrics, smartcards, passwords, time of day, even interaction with
   remote services)
 - using a TPM as a source of entropy for a running OS

> NOTE: At this time this introduction is very much a layman's
> introduction, and only an introduction.  Readers seeking to do
> software development using TPMs will want to make use of [TCG
> specifications and other resources](#Other-Resources).

## Use Cases

Here are _some_ use cases that TPMs can be applied to

 - off-line root of trust measurement (RTM) to check that a device is
   healthy

   ("off-line" meaning "no network needed")

    - encrypted storage

 - online RTM to check that a device is healthy and authorize it to have
   access to a network

   ("online" meaning "demonstrate health via networked interaction with
   other devices")

    - encrypted storage

 - hardware security module (HSM)

    - certification authority (CA)
    - TPMs as smartcards

 - authentication and authorization of console and/or remote user logins

    - e.g., require biometrics, smartcard, admin credentials, multiple
      users' authentication, time-of-day restrictions, etc.

 - entropy source (random number generator)

 - cryptographic co-processor

## Glossary

> For a glossary, see section 4 of [TCG TPM 2.0 Library part 1:
> Architecture](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf).

# Core Concepts

Some core concepts in the world of TPMs:

> NOTE: We will not cover all of these here.

 - cryptography
 - hash extension
 - cryptographic object naming
 - platform configuration registers (PCRs)
 - immutability of object public areas
 - key hierarchies
 - key wrapping
 - restricted cryptographic keys
 - limited resources
 - sessions and authorization
 - other object types, mainly non-volatile (NV) indexes
 - attestation

We'll assume reader familiarity with the basics of cryptography -- the
basics of cryptographic primitives as interfaces, but not their
internals.  E.g., hash functions, symmetric encryption, asymmetric
encryption, and digital signatures.

Authorization is the most important aspect of a TPM, since that's
ultimately what it exists for: to authorize a system or application to
perform certain duties when all the desired conditions allow for it.

TPMs have a very rich set of options for authorization.  It's not just
[policies](#Policies), but also cryptographic object names used with
restricted keys to allow access only to applications that also have
other access.

Where to start?  Let's start with hash extension.

## Hash Extension

Hash extension is just appending some data to a current digest-sized
value, hashing that, and then calling the output the new current value:

```
  v_0 = 0         # size-of-digest-output zero bits
  v_1 = Extend(v_0, e_0)
      = H(v_0 || e_0)
  v_2 = Extend(v_1, e_1)
      = H(v_1 || e_1)
      = H(H(v_0 || e_0) || e_1)
  v_3 = Extend(v_2, e_2)
      = H(v_2 || e_2)
      = H(H(v_1 || e_1) || e_2)
      = H(H(H(v_0 || e_0) || e_1) || e_2)
  ..
  v_n = Extend(v_n-1, e_n-1)
      = H(v_n-1 || e_n-1)
      = H(H(v_n-2 || e_n-2) || e_n-1)
      = H(H(...) || e_n-1)
```

where `H()` is a cryptographic hash function.

Each extension value can be arbitrarily large, and the TPM will use the
traditional `Init`/`Update`/`Final` approach to making digest
computation online.

Note that `H(e0 || e1 || e2) != Extend(Extend(Extend(0, e0), e1), e2)`.
Hash extension makes "message" boundaries strong.

Hash extension is most of what a PCR is, but hash extension is used in
other TPM concepts besides PCRs, such as policy naming.

## Coping with Severe Resource Limits Using Digests and Hash Extension

Hardware TPMs are extremely limited in memory and non-volatile memory
capacity.  As a result they cannot hold large entities.

A common theme in TPMs is the use of digests, and hash extension digests
in particular, as a stand-in for large entities that might not fit at
once on the TPM.

TPMs use digests as stand-ins for large entities of various types:

 - eventlogs
 - policies
 - auditing

We'll discuss at least two of those: event logs, and policies.

## Platform Configuration Registers (PCRs)

A PCR, then, is just a hash extension output.  The only operations on
PCRs are: read, extend, and reset.  All richness of semantics of PCRs
come from how they are used:

 - what the governing TCG platform specification says about them
 - what they are extended with and by what code (in what locality)
 - what purposes they are read for
    - attestation
    - authorization

Note that a PCR value by itself is devoid of meaning.  To add meaning
one must have access to the list of discrete values extended into the
PCR, as well as the order in which they were extended into the PCR.  And
one must know the meaning of each such value.

### Eventlogs

Any TPM-using platform has to provide a way to keep a log of PCR hash
extension values.  Such a log is known as the "eventlog".

The TPM itself cannot hold this log -- the TPM is too
resource-constrained.

## Root of Trust Measurements (RTM)

When a computer and its TPM start up, most PCRs are set to all-zeros,
and then the computer's boot firmware performs a core root of trust
measurement (CRTM) to "measure" (i.e., hash) the the next boot stage and
extend it into an agreed-upon PCR.  The entire boot process should,
ideally, perform RTMs.  At the end of the boot process some set of PCRs
should reflect the totality of the code path taken to complete booting.

Some PCRs are used to measure the BIOS, others to measure option ROMs,
and others to measure the operating system.  Each platform has a
specification for which PCRs are used or reserved for what purposes.  In
principle one could measure the entirety of an operating system and all
the code that is installed on the system.

RTM can be used to ensure that only known-trusted code is executed, and
that important resources are not unlocked unless the state of the system
when they are needed is "has only executed trusted code to get here".

Note that some PCRs are left to be used by "applications".

Some terms:

 - core RTM (CRTM) -- initial measurements performed upon power-on
 - static RTM (SRTM) -- subsequent-to-CRTM measurements of next boot
   stages
 - dynamic RTM (DRTM) -- measurements involved in rebooting

Resource unlocking can be done by creating objects tied to a set of PCRs
such that they must each have specific values for the TPM to be willing
to unlock (unseal) the object.

### The PCR Extension Eventlog

On the "PC platform" (which includes x64 servers) the BIOS keeps a log
of all the PCR extensions it has performed.  The OS should keep its own
log of extensions it performs of PCRs reserved to the OS.  Each
application has to keep a log of the extensions of the PCRs allocated to
it.  Again, the TPM itself cannot do this.

The eventlog documents how each PCR evolved to their current state,
whatever it might be.  Since PCR extension values are typically digests,
the eventlog is very dry, but it can still be used to evaluate whether
the current PCR values represent a trusted state.  For example, one
might have a database of known-good and known-bad firmware/ROM digests,
then one can check that only known-good ones appear in the eventlog and
that reproducing the hash extensions described by the eventlog produces
the same PCR values as one can read, and if so it follows that the
system has only executed trusted code to arrive at the state identified
by the PCRs.

Note though that PCRs and RTM are not enough on their own to keep a
system from executing untrusted code.  A system can be configured to
allow execution of arbitrary code at some point (e.g., download and
execute) and to not extend PCRs accordingly, in which case the execution
of untrusted code will not be reflected in any RTM.

## Tickets

> Tickets are yet another device for coping with TPMs having limited
> resources.  Interaction with TPMs is via request/response
> commands, and tickets are largely about making TPMs stateless between
> related commands.

To avoid having to re-perform various operations -or remember having
performed them- between command invocations, a TPM can produce a
"ticket" that consists of an HMAC over a TPM-generated assertion, keyed
by a key known only to the TPM, and return it to the caller who can then
present it in a subsequent command related to the first.

For example, when signing data the TPM will first digest the data to
sign over several commands and generate a ticket saying it did produce
that digest, then later it can sign the digest after validating the
ticket that it produced.

Another example is a policy ticket, which allows one to avoid having to
re-authenticate (e.g., with smartcard, biometrics, passwords) on every
command for small window of time.

> When would a user be authenticated?  Well, typically at boot time, or
> maybe at wake from sleep/hibernate time.  A laptop could be configured
> to require a user to authenticate with biometrics and possibly a
> password or a smartcard.  Note that such policies are not required by
> the specifications, but rather something that one can choose to use.

> There are five types of tickets.  We won't cover them here.  Readers
> who end up needing to know about them can look at section 11.4.6.3 of
> `TCG TPM 2.0 Library, part 1: Architecture`.

## Cryptographic Object Naming

TPMs support a variety of types of objects.  Objects generally have
pointer-like "handles" that they are often used in the TPM APIs.  But
more importantly, objects have cryptographically-secure names that are
used in many cases.

  The cryptographically-secure name of an object is the hash of the
  object's "public area".

The public area of, say, an asymmetric key is a data structure that
includes the public key (corresponding to the private key), and various
attributes of the key (e.g., its algorithm, but also whether it is bound
to the TPM where it resides, or to its key hierarchy), unseal policy,
and access policy.

This concept is extremely important.  Because object names are the
outputs of cryptographically strong digest (hash) functions, they are
resistant to collision attacks, first pre-image attacks, and second
pre-image attacks -- as strong as the hash algorithm used anyways.
Which means that object names cannot be forged easily, which means that
they can be used in context where a peer needs certain guarantees, or
to defeat active attacks.

### Immutability of Object Public Areas

Because the name of an object is a digest of its public area, the public
area cannot be changed after creating it.  One can delete and then
recreate an object in order to "change" its public area, but this
necessarily yields a new name.

### Cryptographic Object Naming as a Binding

> This section comes too soon, since it relates to attestation and
> restricted keys.  Still, it may be useful to illustrate cryptographic
> object naming with one particularly important use of it.

A pair of functions,
[`TPM2_MakeCredential()`](/TPM-Commands/TPM2_MakeCredential.md) and
[`TPM2_ActivateCredential()`](/TPM-Commands/TPM2_ActivateCredential.md),
illustrate the use of cryptographic object naming as a binding or a sort
of authorization function.

[`TPM2_MakeCredential()`](/TPM-Commands/TPM2_MakeCredential.md) can be
used to encrypt a datum (a "credential") to a target TPM such that the
target will _only be willing to decrypt it_ if *and only if* the
application calling `TPM2_ActivateCredential()` to decrypt that
credential has access to some key named by the sender, and that name is
a cryptographic name that the sender can and must compute for itself.

The semantics of these two functions can be used to defeat a
cut-and-paste attack in attestation protocols.

## Key Hierarchies

TPMs have multiple key hierarchies, each with zero, one or more primary
keys, each with zero, one, or more children keys:

```
                seed
                /|\
               / | \
              v  v  v
     primary key (asymmetric encryption)
                /|\
               / | \
              v  v  v
       secondary keys (of any kind)
                /|\
               / | \
              v  v  v
                ...
```

Keys that have no parent are primary keys.

There are four built-in hierarchies:

 - platform hierarchy
 - endorsement hierarchy
 - storage hierarchy
 - null hierarchy

of which only the endorsement and storage hierarchies will be of
interest to most readers.

The endorsement hierarchy is used to authenticate (when needed) that a
TPM is a legitimate TPM.  The primary endorsement key is known as the EK
(endorsement key).  Hardware TPMs come with a certificate for the EK
issued by the TPM's manufacturer.  This EK certificate ("EKcert") can be
used to authenticate the TPM's legitimacy.  The EK's public key
("EKpub") can be used to uniquely identify a TPM, and possibly link to
the platform's, and even the platform's user(s)' identities.

The [`TPM2_CreatePrimary()`](TPM2_CreatePrimary.md) command creates
primary key objects deterministically from the hierarchy's seed and the
"template" used to create the key (which includes a "unique" area that
provides "entropy" to the key derivation function).

The [`TPM2_Create()`](TPM2_Create.md) command creates a ordinary
objects.

The [`TPM2_CreateLoaded()`](TPM2_CreateLoaded.md) command can also
create primary key objects deterministically from the hierarchy's seed
and the "template" used to create the key (which includes a "unique"
area that provides "entropy" to the key derivation function).

## Key Wrapping and Resource Management

Key wrapping is encrypting a secret or private key (key encryotion key,
or KEK) such that a specific entity may recover the plain key.

A decrypt-only asymmetric private key can be used to encrypt secrets to
the TPM on which that private key resides.

As well as wrapping secrets by encryption to public keys, TPMs also use
wrapping in a symmetric key known only to the TPM for the purpose of
saving keys off the TPM.

This is used for resource management: since hardware TPMs have very
limited resources, objects need to created or loaded, used, then saved
off-TPM to make room for other objects to be loaded (unless they are not
to be used again, then saving them is pointless).  Only a TPM that saved
an object can load it again, but some objects can be exported to other
TPMs by encrypting them to their destination TPMs' EKpubs.

With a resource manager and access broker, a TPM can appear to have
infinite resources.

### Controlling Exportability of Keys

A key that is `fixedTPM` cannot leave the TPM in cleartext.  It can be
saved off the TPM it resides in, but only that TPM can load it again.

A key that is `fixedParent` cannot be moved from one part of a key
hierarchy to another -- it cannot be "re-parented".  Though if its
parent is neither `fixedParent` nor `fixedTPM` then the parent and its
descendants can be moved as a group to some other TPM.

> Key hierarchies are an important TPM topic that we're not really
> addresing in this intro.

## Persistence

In a TPM, key objects are, by default, transient, meaning the TPM will
forget them if restarted.  Still, they can be saved (encrypted in a
secret key only the TPM knows) and later reloaded.

Transient objects can be made persistent, but because hardware TPMs have
very little non-volatile memory, few keys should be made persistent.
Instead you can save them (wrapped to a TPM-only KEK) and reload them as
needed.

Because primary keys (for any hierarchy other than the null hierarchy)
are derived deterministically from a built-in and protected seed, and
from a template, they are persistent even when not moved to NV storage
and even when not saved as long as the hierarchy's seed is not reset.

(Resetting the endorsement hierarchy seed is a very dramatic action, as
it changes the EK/EKpub and renders any provisioned EKcert useless.
Resetting the storage hierarchy seed is much less dramatic.  The NULL
hierarchy is reset every time the TPM resets.)

PCRs always persist, but they get reset on restart.

NV indexes always persist.  (But in disorderly resets/shutdowns a
hybrid NV index may not be sync'ed to NV.)

## Non-Volatile (NV) Indexes

TPMs also have a special kind of non-volatile object: NV indexes.

> NOTE: NV indexes are not "objects" in the sense that the TCG's
> specifications mean.  TCG's definition of "object" is
>
> > key or  data that  has  a public  portion  and, optionally, a
> > sensitive  portion;  and which is  a  member of a hierarchy

NV indexes come in multiple flavors for various uses:

 - store public data (e.g., an NV index is used to store the EKcert)
 - emulate PCRs
 - monotonic counters
 - fields of write-once bits (bitfields) (for, e.g., revocation)
 - ...

NV indexes can be used standalone, and/or in connection with policies,
to enrich application TPM semantics.

## Authentication and Authorization

TPMs have multiple ways to authenticate users/entities:

 - plain passwords (legacy)
 - HMAC based on secret keys or passwords
 - public key signed attestations of identification by biometric
   authentication devices

TPMs have two ways to authorize access to various objects:

 - plain passwords (legacy)
 - HMAC proof of possession of a secret key or password
 - arbitrarily complex policies that can make reference to:
    - PCR state
    - NV index state
    - time of day
    - authentication state
    - etc.

### Policies

A policy consists of a sequence of "commands" that each asserts
something of interest.

Policies are particularly interesting because they are cryptographically
named using hash extension with the sequence of "commands" that make up
a policy.  Therefore no matter how complex and large a policy is, it is
always "compressed" to a hash digest.

It is the responsibility of the application that will attempt to use a
policy-protected resource to know what the policy's definition is and
restate it to the TPM when it goes to make use of that resource.  The
TPM will evaluate the policy and, at the end, check that its digest
matches that of the policy-protected resource.  Thus, and because policy
digests are small and fixed-sized, they can be arbitrarily more complex
than a TPM's limited resources would otherwise allow.

All the policy commands that are to be evaluated successfully to grant
access have to be known to the entity that wants that access.  Of
course, that entity will have to satisfy -at access time- the conditions
expressed by the relevant policy.  And that entity has to know the
policy because the TPM knows only a digest of it.

### Policy Construction

Construction of a policy consists of computing it by hash extending an
initial all-zeroes value with the commands that make up the policy.

This can be done entirely in software, but the TPM supports a notion of
"trial sessions" where the application can issue policy commands to
build up a policy digest without the application having to know how to
do that in software.  Trial sessions have every policy command succeed,
but they authorize nothing -- the point of a trial session is only to
compute and extract a `policyDigest` at the end of the policy.

### Policy Evaluation

Evaluation of a policy consists of issuing those same commands to the
TPM in a [non-trial] session, with those commands either evaluated
either immediately or deferred to the time of execution of the
to-be-authorized command, but the TPM computes the same hash extension
as it goes.  Once all policy commands issued have been evaluated and
have succeeded, the resulting hash extension value is compared to the
policy that protects the resource used by the to-be-authorized command,
and if and only if the digest matches, then the command is allowed,
otherwise it is not.

For example, one might construct a policy like so:

```bash
$ tpm2 flushcontext -t
$ tpm2 flushcontext -s
$ tpm2 startauthsession --session session.ctx --policy-session
$ tpm2 policysecret --session session.ctx --object-context endorsement
$ tpm2 policycommandcode -S session.ctx -L activate.ctx TPM2_CC_ActivateCredential
```

which saves the digest of the policy in a file named `activate.ctx`.

### Indirect Policies

Because an object's policy is part of its name, that policy cannot be
changed after creation.  An indirect policy command allows for a policy
to change over time without having to recreate the authorized object.

### Compound Policies

Policies consist of a conjunction (logical-AND) of assertions that must
be true at evaluation time.

However, there is a special policy command that allows for alternation
(logical-OR).  This policy command has a number of alternative policy
digests.  At evaluation time, one of the alternation digests must match
the extension value for the policy commands up to (but excluding) the
logical-OR policy command.  At evaluation time the caller must have
picked one of the alternatives and executed the commands that make it
up.

(Recall that the application has to know the definition of the policy
because the TPM knows only the policy's digest.)

### Rich Policy Semantics

With all these features, and with all the flexibility allowed by NV
indexes, policies can be used to:

 - require that N-of-M users authenticate
 - require multi-factor authentication (password, biometric, smartcard)
 - enforce bank vault-like time of day restrictions
 - check revocation (using NV index bit-field objects)
 - check system RTM state (PCRs)
 - distinguish user roles

## Sessions

A session is an object (meaning, among other things, that it can be
loaded and unloaded as needed) that represents the current state used
for authorization of actions or for encryption of traffic between the
application and the TPM.

There are two types of sessions then: those used for authorization, and
those used for encryption of application `<->` TPM communication.

Authorization sessions contain state such as a `policyDigest`
representing authorization policy that has been satisfied, and various
other state.  TPM commands may check that an authorization session's
state satisfies the requirements for use of the argument objects passed
to the commands.

> NOTE: Every command input parameter that is a handle that requires
> authorization must have its own session associated with it.

### Authorization Session State

Authorization sessions have a number of state attributes.  Some of these
are set at the time of creation of the session.  Some of these can be
set directly with `TPM2_Policy*()` commands.  Others evolve in other
ways.  These state attributes are:

 - `policyDigest`

   A hash extension digest of all the policy commands sent by the
   application in this session thus far.  Every successful
   `TPM2_Policy*()` command extends this.

   Objects may have a policy digest set on them to refer to the policy
   that an application must meet in order to use them.  The application
   has to issue the `TPM2_Policy*()` commands, in order, that produce
   that digest, the commands must all succeed, and the `policyDigest`
   must equal that of the object.

 - `isTrialPolicy`

   When this is set then the session will not authorize anything at all
   and all policy commands will be assumed to be met and will not be
   evaluated.  This is useful for constructing and extracting from the
   TPM the `policyDigest` of a policy to set on some future new
   object(s).

   Sessions that have this set are known as "trial sessions".

   Applications can construct `policyDigest` values entirely in
   software, but using the TPM with a trial session saves one the
   bother.

 - `commandCode`

   Identifies a command that will be authorized by the policy referred
   to by `policyDigest`.

   If a policy requires this, then it authorizes the one command
   identified by the command code.

 - `cpHash`

   A hash of some command's parameters.  If a policy requires this, then
   it authorizes the one command whose parameters match this hash.

 - `commandLocality`

   A locality that the application must be in.

 - policy reuse / expiration state:

    - `startTime`

      The start time of the session.

    - `timeout`

      The lifetime of the session.

    - `nonceTPM`

 - Authentication requirements:

    - `isAuthValueNeeded`

    - `isPasswordNeeded`

    - `isPPRequired`

      PP == physical presence.

 - `checkNvWritten`

 - `nvWrittenState`

 - `nameHash`

 - `pcrUpdateCounter`

### Encryption Sessions

Sessions can also be used for encrypting TPM command arguments and
results.  This can be useful when one does not trust the path to the
TPM, such as when the TPM is remote.

Only the first input parameter of a TPM command will be encrypted, and
only the first output parameter of a TPM command will be encrypted, and
that only if when that first parameter is of type `TPM2B`.  Those first
`TPM2B` type command input and/or output parameters will be encrypted
with a symmetrict AES key derived from a secret key established via RSA
key transport or ECDH key agreement.

> Encryption sessions are useful for when the path to a TPM is not
> trused, such as when a TPM is a remote TPM, or when otherwise the path
> to the TPM is not trusted.

### Key Exchange for Encryption Sessions

> Encryption sessions are useful for when the path to a TPM is not
> trused, such as when a TPM is a remote TPM, or when otherwise the path
> to the TPM is not trusted.  This section talks about key exchange for
> such situations.

The symmetric keys used for TPM command encryption are exchanged at
session creation time.

Keys can be provided by one of either RSA key transport or ECC key
agreement, and/or the secret `authValue` of a loaded entity.

Sessions are created with
[`TPM2_StartAuthSession()`](/TPM-Commands/TPM2_StartAuthSession.md),
which has serveral _optional_ input and output parameters related to
session encryption.  In particular it provides ways to create a session
key for command parameter encryption:

 - RSA key transport

   The caller can encrypt a secret to a public key for which the TPM has
   loaded the private key as identified by the `tpmKey` input parameter
   of [`TPM2_StartAuthSession()`].

 - ECDH key agreement

   The caller can generate an ephemeral ECDH key and use it with the
   public key of the ECDH key object identified by the `tpmKey` input
   parameter of [`TPM2_StartAuthSession()`].  The TPM will use the
   private key of the object identified by the `tpmKey` input parameter
   and the ephemeral public key sent by the caller in the
   `encryptedSalt` input parameter.

 - use the `authHash` of the entity identified by the `bind` input
   parameter

The caller computes the same session key as the TPM.

To authenticate the TPM and prevent active attacks, the caller of
`TPM2_StartAuthSession()` should use an `EK` as the `tpmKey` and its
`EKpub` to locally compute the session key.  Alternatively the caller
can use a non-`EK` key object created over an earlier encrypted session
that authenticated the target TPM.

> A non-null `bind` parameter can be used to create a "bound" session
> that can be used to satisfy HMAC-based authorization for specific
> objects.  We will not cover this in detail here.

### HMAC Sessions

An HMAC session proves the caller knows the `authValue` secret of some
entity.  This functions a lot like a password, with the `authValue`
used to compute HMACs that prove possession, but with the `authValue`
generally being large and randomly generated, thus much stronger than a
password.

Typically the `authValue` of some entity should be sent encrypted to the
TPM when creating an entity, with [the encrypted session being keyed via
RSA key transport or ECDH](#Key-Exchange-for-Encryption-Sessions).  This
way an `authValue`, though a simple, password-like binary string, can be
strong and secure due to being large, randomly chosen and sent over
an encrypted session.

### Password Sessions

> WIP [Say something about passwords and password sessions, besides
> "don't use them" and "support remains mostly for TPM 1.2 reasons".]

Alternatively a session can be for encryption of command inputs/outputs,
which is useful when the path to the TPM is not secure.

### Authorization Roles

How a TPM authorizes some particular command and its use of its input
handles varies by command and according to two attributes of the objects
identified by the command's input handles.

A "role" is really a set of rules that will be applied to authorization
for a given object/command.

There are three types of authorization roles that can apply in any case:

 - `USER` role

   This means that if the object in question has an `authValue` set, and
   it has the `userWithAuth` attribute set, then the application can get
   access by demonstrating knowledge of the corresponding password.  And
   anyways, if the object has a policy then the user can get access by
   satisfying the policy even w/o knowing the password.

   This is the most commonly applied authorization role.

 - `ADMIN` role

   This means that if the object in question has an `authValue` set, and
   it has the `adminWithPolicy` attribute _not_ set, then the
   application can get access by demonstrating knowledge of the
   corresponding password or by satisfying its `authPolicy` if one is
   set.  But if the object has a policy and the `adminWithPolicy`
   attribute set then the user _must_ satisfy the policy to get access.

   In the `adminWithPolicy` attribute set case, the caller _must_ also
   have called `TPM2_PolicyCommandCode()` with the code of the command
   that the caller wishes to execute.

   Only three commands apply `ADMIN` role to any of the objects
   identified by their input handle parameters:

    - `TPM2_Certify()` requires `ADMIN` role for its `objectHandle`
      input parameter.

      Whereas the `signHandle` input parameter requires `USER` role.

    - `TPM2_ActivateCredential()` requires `ADMIN` role for its
      `activateHandle` input parameter.

      Whereas the `keyHandle` input parameter requires `USER` role.

    - `TPM2_ObjectChangeAuth()` requires `ADMIN` role for its
      `objectHandle` input parameter.

 - `DUP` role

   This is just for the
   [`TPM2_Duplicate()`](/TPM-Commands/TPM2_Duplicate.md) command.  The
   caller of `TPM2_Duplicate()` must satisfy the key object's
   `authPolicy`, and must have called `TPM2_PolicyCommandCode()` with
   the code of the `TPM2_Duplicate()` command (`TPM_CC_Duplicate`).

   > `DUP` is very similar to `ADMIN` when the `adminWithPolicy`
   > attribute is set.

For example, the
[`TPM2_ActivateCredential()`](/TPM-Commands/TPM2_ActivateCredential.md)
command requires `USER` role for the `keyHandle` input and `ADMIN` role
for the `activateHandle`.  If the `keyHandle` is the `EK`, then since
the `EK` has a default `authValue`, use will be allowed.  If the
`activateHandle` is for an object with an `authPolicy` and the
`adminWithPolicy` attribute set, then the caller must execute that
policy's commands (yielding, on success, a session whose `policyDigest`
matches that object's `authPolicy`) and must have called
`TPM2_PolicyCommandCode(TPM_CC_ActivateCredential)` on that same
session.

> NOTE: Every handle argument to a TPM command can require its own
> authorization, therefore there can be zero, one, or two authorization
> sessions as inputs to any TPM command (some TPM commands have no input
> handle parameters, some have one, and some have two).

### Proving Knowledge of a Shared `authValue`

Each session used in a command can prove the caller's (and TPM's)
knowledge of the session's `bind` object's `authValue`.  As well, the
`TPM2_PolicySecret()` command can be used to inject the `authValue` of
arbitrary objects into the session's state computations, once again
having the caller prove its knowledge of the `authValue`, and the TPM
will prove its knowledge of it back to the caller on its response.

## Restricted Cryptographic Keys

Cryptographic keys can either be unrestricted or restricted.

An unrestricted signing key can be used to sign arbitrary content.

An unrestricted decryption key can be used to decrypt arbitrary
ciphertexts encrypted to that key's public key.

> NOTE WELL: The endorsement key (EK) is a restricted key.

### Restricted Signing Keys

A restricted signing key can be used to sign only TPM-generated content
as part of specific TPM restricted signing commands.  Such content
always begins with a magic byte sequence.  Conversely, the TPM refuses
to sign externally generated content that starts with that magic byte
sequence.  See the [`TPM2_Certify()`](/TPM-Commands/TPM2_Certify.md),
[`TPM2_Quote()`](/TPM-Commands/TPM2_Quote.md), `TPM2_CertifyCreation()`,
`TPM2_GetSessionAuditDigest()`, and `TPM2_GetCommandAuditDigest()` TPM
commands.

There is also a notion of signing keys that can only be used to sign
PKIX certificates using `TPM2_CertifyX509()`.

### Restricted Decryption Keys

> NOTE WELL: The endorsement key (EK) is a restricted key.

A restricted decryption key can only be used to decrypt ciphertexts
whose plaintexts have a certain structure.  In particular these are used
for [`TPM2_MakeCredential()`](/TPM-Commands/TPM2_MakeCredential.md) /
[`TPM2_ActivateCredential()`](/TPM-Commands/TPM2_ActivateCredential.md)
to allow the TPM-using application to get the plaintext if and only if
(IFF) the plaintext cryptographically names an object that the
application has access to.  This is used to remotely communicate secrets
("credentials") to TPMs.

Another operation that a restricted decryption key can perform is
[`TPM2_Import()`](/TPM-Commands/TPM2_Import.md), which decrypts a key
wrapped to the given decrypt-only key and outputs a file that can be
loaded with [`TPM2_Load()`](/TPM-Commands/TPM2_Load.md).  The wrapped
key payload given to [`TPM2_Import()`](/TPM-Commands/TPM2_Import.md) too
has a particular structure and is produced by a remote peer using
[`TPM2_Duplicate()`](/TPM-Commands/TPM2_Duplicate.md).

To recap, a restricted decryption key can only be used to:

 - "activate credentials" (made with
   [`TPM2_MakeCredential()`](/TPM-Commands/TPM2_MakeCredential.md))

 - receive wrapped keys sent by a peer (made with
   [`TPM2_Duplicate()`](/TPM-Commands/TPM2_Duplicate.md))

## Attestation

Attestation is the process of demonstrating that a system's current
state is "trusted", or the truthfulness of some set of assertions.

Often a system gets something in exchange for attesting to its current
state.  E.g., keys for unlocking filesystems, or device credentials.

As you can see in our [tutorial on attestation](/Attestation/README.md),
many TPM concepts can be used to great effect:

 - using PCRs to attest to system state
 - using policies and sealed-to-PCRs objects to attest to authentication
   events on the system
 - using restricted keys and cryptographic object naming to authenticate
   a TPM and bind it to its host
 - delivering key material to authenticated systems via their TPMs
 - unlocking resources following successful attestation
 - authorization of devices onto a network
 - etc.

## Use Cases (reprise)

### Off-line RTM / TOTP

Use a TPM to generate a time-based one-time (TOTP) password based on
current time and a seed derived from selected PCR values, then display
this TOTP.  A user can then check that the TOTP presented by the device
matches the TOTP on a separate authenticator.

Links:

 - https://github.com/tpm2-software/tpm2-totp
 - https://github.com/mjg59/tpmtotp
 - https://trmm.net/Tpmtotp/

### Online RTM (aka Attestation)

See [our tutorial on attestation](/Attestation/README.md).

### Encrypted Storage

 - [Safeboot](https://safeboot.dev/)
 - [Hacking with a TPM](https://c3media.vsos.ethz.ch/congress/2019/slides-pdf/36c3-10564-hacking_with_a_tpm.pdf)

### HSM / CA / Smartcard

Use `TPM2_Sign()` or `TPM2_CertifyX509()` to sign certificates with a
TPM-resident key that is fixedTPM and fixedParent.

Use `TPM2_GetCommandAuditDigest()` to implement an audit trail for the
CA.

### Authentication and Authorization of Console and/or Remote User Logins

Use [TPM policies](#Authentication-and-Authorization).

### Entropy Source

See our tutorial on [TPM-based RNGs](/Random_Number_Generator/README.md).

### Cryptographic Co-Processor

Use cryptographic primitives provided by the TPM using unrestricted key
objects:

 - use TPM cryptographic primitives commands directly -- see
   [TCG TPM 2.0 Library part 3: Commands, sections 14 and 15](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf)

 - use PKCS#11 with a TPM-backed token provider:
    - https://github.com/tpm2-software/tpm2-pkcs11
    - https://docs.oracle.com/cd/E36784_01/html/E37121/gmsch.html
    - https://incenp.org/notes/2020/tpm-based-ssh-key.html
    - http://trousers.sourceforge.net/pkcs11.html
    - https://www.lorier.net/docs/tpm

 - use OpenSSL with a PKCS#11 `ENGINE` (see above)

 - use OpenSSL with a TPM `ENGINE`
    - https://github.com/tpm2-software/tpm2-tss-engine

# Other Resources

 - [A Practical Guide to TPM 2.0](https://trustedcomputinggroup.org/resource/a-practical-guide-to-tpm-2-0/)
   is an excellent book that informed much of this tutorial.

 - Of course, there is the [TPM.dev community](https://developers.tpm.dev/),
   which has many resources, posts, a chat room, and knowledgeable
   participants.

 - Nokia has a [TPM course](https://github.com/nokia/TPMCourse/tree/master/docs).

 - [Hacking with a TPM](https://c3media.vsos.ethz.ch/congress/2019/slides-pdf/36c3-10564-hacking_with_a_tpm.pdf).

 - [Microsoft has solid TPM resources](https://docs.microsoft.com/en-us/windows/security/information-protection/tpm/trusted-platform-module-top-node).

 - The TCG has a number of members-only tutorials, but it seems that it
   is possible to be invited to be a non-fee paying member.

 - Core TCG TPM specs:

    - [TCG TPM 2.0 Library part 1: Architecture](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf).
    - [TCG TPM 2.0 Library part 2: Structures](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part2_Structures_pub.pdf).
    - [TCG TPM 2.0 Library part 3: Commands](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf).
    - [TCG TPM 2.0 Library part 3: Commands Code](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_code_pub.pdf).
