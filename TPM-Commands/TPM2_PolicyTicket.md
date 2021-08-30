# `TPM2_PolicyTicket()`

`TPM2_PolicyTicket()` is very similar to
[`TPM2_PolicySigned()`](TPM2_PolicySigned.md), except that a
TPM-produced ticket is used instead of a signature made by some entity,
and the TPM acts as though the [`TPM2_PolicySigned()`](TPM2_PolicySigned.md)
or [`TPM2_PolicySecret()`](TPM2_PolicySecret.md) command used to produce
the ticket had been executed instead of `TPM2_PolicyTicket()`.

This is useful for avoiding excessive interactions with a user in a
short period of time.  E.g., prompting the user at most once every so
many minutes for:

 - a password,
 - smartcard PIN entry,
 - and/or biometrics identification.

## Inputs

 - `TPMI_SH_POLICY policySession` (handle to the session being extended)
 - `TPM2B_DIGEST cpHashA` (the command parameter hash of a single command to be authorized, or `Empty Buffer` to not so-limit the assertion)
 - `TPM2B_NONCE policyRef` (an opaque value of the caller's and/or signer's choosing that is used to limit the value of the signature and to extend the `policySession`'s `policyDigest` along with the `authObject`'s name)
 - `TPM2B_NAME authName` (the name of the object used in the `TPM2_PolicySigned()` or `TPM2_PolicySecret()` command that produced the `ticket`)
 - `INT32 expiration` (a positive or negative number of milliseconds which, if non-zero, sets an expiration for this assertion; if zero or positive then a `policyTicket` will not be output)
 - `TPMT_TK_AUTH ticket` (the ticket)

## References

 - [TCG TPM Library part 1: Architecture, section 19.7.12](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 1: Architecture, section 19.7.15](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 3: Commands, section 23.3](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf)

