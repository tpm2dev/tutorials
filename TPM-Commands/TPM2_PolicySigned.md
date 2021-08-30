# `TPM2_PolicySigned()`

`TPM2_PolicySigned()` allows a caller to provide a signature of some
assertion, with the signature made by some authorizing entity identified
by its public key.

The semantics of the signature are arbitrary and up to the entity
constructing the policies that contain this command.  Possible semantics
include:

 - biometrics user identification (whereby the biometrics device signs
   an assertion that a user identified by the opaque value of
   `policyRef` has been identified biometrically)

 - smartcard-based user authentication (whereby access to a private
   signing key on a smartcard is used to sign an assertion that the user
   has been authenticated by the user's possession of the smartcard and
   interactive PIN entry to unlock it)

 - assertion of attested state being trusted (whereby an attestation
   server signs such an assertion)

 - etc.

The signature made by the signed is over the following digest:

  `aHash := H(nonceTPM || expiration || cpHashA || policyRef)`

where `H()` is the digest algorithm associated with the authorizing
entity's public key.

When evaluating this assertion in a policy session, the TPM will check
that the signature matches the above hash as constructed by the TPM from
the `TPM2_PolicySigned()` command parameters.

When evaluating this assertion in a trial session, the TPM will ignore
the signature and will extend the `policySession`'s `policyDigest` as if
the signature had matched the hash.

The `nonceTPM` input strongly binds the command to the `policySession`.
If the `Empty Buffer` is given as the `nonceTPM`, then the
`TPM2_PolicySigned()` command could be altered to refer to any other
policy any TPM.  For this reason it is important to use the
`policySession`'s `nonceTPM` in any call to `TPM2_PolicySigned()`.

If a `policyTicket` is requested and output, that ticket can be used (up
to its `expiration`), via `TPM2_PolicyTicket()`, to satisfy the same
`TPM2_PolicySigned()` that produced the ticket.

That is, a caller can get a ticket from a `TPM2_PolicySigned()`
invocation that allows it to re-use the `auth` signature many times
prior to the ticket's expiration without having to get the authorizing
entity to re-sign.  For example, if the authorizing entity is a
biometrics identification device, or a smartcard, then the interactive
human identification or interactive smartcard PIN entry steps can be
eschewed by the caller up to the ticket's expiration, using the `ticket`
(via `TPM2_PolicyTicket()`) instead to satisfy the same
`TPM2_PolicySigned()` command in any policy on that TPM.  This is useful
to avoid requiring repeated biometrics or PIN entry in a short time
span.

## Inputs

 - `TPMI_DH_OBJECT authObject` (handle to the key object whose public key is the signing entity's)
 - `TPMI_SH_POLICY policySession` (handle to the session being extended)
 - `TPM2B_NONCE nonceTPM` (the policy nonce for the `policySession`)
 - `TPM2B_DIGEST cpHashA` (the command parameter hash of a single command to be authorized, or `Empty Buffer` to not so-limit the assertion)
 - `TPM2B_NONCE policyRef` (an opaque value of the caller's and/or signer's choosing that is used to limit the value of the signature and to extend the `policySession`'s `policyDigest` along with the `authObject`'s name)
 - `INT32 expiration` (a positive or negative number of milliseconds which, if non-zero, sets an expiration for this assertion; if zero or positive then a `policyTicket` will not be output)
 - `TPMT_SIGNATURE auth` (the signature; ignored if the `policySession` is a trial session)

## Outputs

 - `TPM2B_TIMEOUT timeout` (implementation-specific indication of actual timeout for the session)
 - `TPMT_TK_AUTH policyTicket`

## References

 - [TCG TPM Library part 1: Architecture, section 19.7.12](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 1: Architecture, section 19.7.15](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 3: Commands, section 23.3](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf)

