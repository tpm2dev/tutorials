# `TPM2_PolicySecret()`

`TPM2_PolicySecret()` allows a caller to assert and prove knowledge of
the `authValue` associated with some entity.  The `authValue` affects
the HMAC calculation for the affected session, so the command will fail
if the caller does not know the `authValue`.

The `tpmNonce` input strongly binds the command to the `policySession`.
If the `Empty Buffer` is given as the `tpmNonce`, then the
`TPM2_PolicySecret()` command could be altered to refer to any other
policy on the same TPM if the object referred to by `authHandle`
requires an HMAC or policy session, or any TPM otherwise.

If a `policyTicket` is requested and output, that ticket can be used (up
to its `expiration`), via `TPM2_PolicyTicket()`, to satisfy the same
`TPM2_PolicySecret()` that produced the ticket.

That is, a caller can get a ticket from a `TPM2_PolicySecret()`
invocation that allows it to re-use the `authValue` proof many times
prior to the ticket's expiration without having to actually prove the
`authValue` again.  For example, if the `authValue` is obtained from a
password prompt and the password and `authValue` erased from memory as
soon as the `TPM2_PolicySecret()` command is marshalled, then the caller
can keep satisfying policies containing that `TPM2_PolicySecret()` by
using `TPM2_PolicyTicket()` instead of `TPM2_PolicySecret()`.  This is
useful to avoid requiring repeated password prompts in a short time
span.

## Inputs

 - `TPMI_DH_OBJECT authHandle` (handle to the entity whose `authValue` is to be proven)
 - `TPMI_SH_POLICY policySession` (handle to the session being extended)
 - `TPM2B_NONCE tpmNonce` (the policy nonce for the `policySession`)
 - `TPM2B_DIGEST cpHashA` (the command parameter hash of a single command to be authorized, or `Empty Buffer` to not so-limit the assertion)
 - `TPM2B_NONCE policyRef` (an opaque value of the caller's choosing, possibly the `Empty Buffer`, that is used to extend the `policySession`'s `policyDigest` along with the name of `authHandle`)
 - `INT32 expiration` (a positive or negative number of milliseconds which, if non-zero, sets an expiration for this assertion; if zero or positive then a `policyTicket` will not be output)

## Outputs

 - `TPM2B_TIMEOUT timeout` (implementation-specific indication of actual timeout for the session)
 - `TPMT_TK_AUTH policyTicket`

## References

 - [TCG TPM Library part 1: Architecture, section 19.7.12](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 1: Architecture, section 19.7.15](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture_pub.pdf)
 - [TCG TPM Library part 3: Commands, section 23.4](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf)

