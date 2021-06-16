# `TPM2_StartAuthSession()`

This command starts a session that can be used for authorization and/or
encryption.

## Inputs

 - `TPMI_DH_OBJECT+ tpmKey`

   This optional _input_ parameter specifies the handle of a loaded RSA
   decryption key or of a loaded ECDH key.

 - `TPMI_DH_ENTITY+ bind`

   This parameter, if not null, references a loaded entity whose
   `authValue` will be used in the session key computation.

 - `TPM2B_NONCE nonceCaller`

   This is a nonce chosen by the caller.

 - `TPM2B_ENCRYPTED_SECRET encryptedSalt`

   This optional _input_ parameter must be present if `tpmKey` is
   present.

   If `tpmKey` is an RSA decryption key then `encryptedSalt` must be an
   RSA OEAP ciphertext that will be decrypted with the `tpmKey`.  The
   plaintext will be used to derive symmetric AES-CFB encryption keys.

   If `tpmKey` is an ECDH key, then `encryptedSalt` must be a public key
   that will be used to generate a shared secret key from which
   symmetric AES-CFB encryption keys will be derived.

 - `TPM_SE sessionType`
 - `TPMT_SYM_DEF+ symmetric`
 - `TPMI_ALG_HASH authHash`

   A hash algorithm for the key derivation function.

## Outputs (success case)

 - `TPMI_SH_AUTH_SESSION sessionHandle`
 - `TPM2B_NONCE nonceTPM`

   This is an _output_ parameter that is generated by the TPM, and it is
   a nonce that is used for key derivation.

## Session Types

The `sessionType` input parameter must be one of:

 - `TPM_SE_HMAC`
 - `TPM_SE_POLICY`
 - `TPM_SE_TRIAL`

### HMAC Sessions

If the session is to be an HMAC session authenticating knowledge of some
entity's `authValue`, then the `bind` argument must be provided.

### Authorization Sessions

For policy sessions, the caller should now call one or more
`TPM2_Policy*()` commands to execute the policy identified by the
`authPolicy` value of the entity to be accessed via this session.

### Trial Policies

For trial sessions, the caller should now call one or more
`TPM2_Policy*()` commands as will be used in future actual policy
sessions, then extract the `policyDigest` of the
session after the last policy command -- that will be a value
suitablefor use as an `authPolicy` value for TPM entities.

### Encryption Sessions

> All sessions can be used for encryption that were created with either
> or both of the `bind` input parameter and the pair of input parameters
> `tpmKey` and `encryptedSalt` set.

> Encryption sessions are useful for when the path to a TPM is not
> trused, such as when a TPM is a remote TPM, or when otherwise the path
> to the TPM is not trusted.  This section talks about key exchange for
> such situations.

For encryption sessions the `symmetric` parameter should also be set.

Encryption sessions can have a session key derived from either or both
of the `authValue` of the `bind` entity or the key exchange represented
by the `tpmKey` and `encryptedSalt` inputs.  The `nonceCaller` input and
the `nonceTPM` output will also salt the key derivation.

The symmetric keys used for TPM command encryption are set at session
creation time.

Session keys are derived from the `tpmKey` and `encryptedSalt` inputs
(if provided) and the `authValue` of a loaded entity referred to by
`bind` (if provided), along with the nonces and other things.

The `tpmKey` and `encryptedSalt` inputs can inject a secret either via
RSA key transport or elliptic curve Diffie-Hellman (ECDH).

The key will be derived as follows:

 - if `tpmKey` and `encryptedSalt` are provided, then the key is
   recovered (RSA case) or computed (ECDH case)

   In the RSA case the `encryptedSalt` is the ciphertext resulting from
   RSA PKCS#2 (OEAP) encryption to the public key of the object referred
   to by `tpmKey`.  The TPM will decrypt the `encryptedSalt` to recover
   the secret.

   In the ECDH case the `encryptedSalt` is an ephemeral public key, and
   the secret will be the ECDH shared secret constructed from that key
   and the private part of the `tpmKey`.

 - then the internal `KDFa()` function will invoked as follows:

   ```
   sessionKey := KDFa(authHash,
                      (bind.authValue || tmpKey.get(encryptedSalt)),
                      "ATH",
                      nonceTPM,
                      nonceCaller,
                      authHash.digestSize)
   ```

   where:

    - `bind.authValue` is the `authValue` of the `bind` entity (if
      provided)

    - `tmpKey.get(encryptedSalt)` is the result of RSA decryption of
      `encryptedSalt` if `tpmKey` is an RSA key, or the result of ECDH
      key agreement between the private part of `tpmKey` and the public
      ECDH key in `encryptedSalt` if `tpmKey` is an ECDH key

To avoid active attacks, one would use the EK as the `tpmKey` input
parameter of `TPM2_StartAuthSession()`.  Or one could use a `tpmKey`
input created with, e.g., `TPM2_Create()` over another encrypted session
that itself used the EK as its `tpmKey` input.

> A non-null `bind` parameter can be used to create a "bound" session
> that can be used to satisfy HMAC-based authorization for specific
> objects.  We will not cover this in detail here.

## References

 - [TCG TPM Library part 1: Architecture, sections 18.6, 19, and 21](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part1_Architecture.pdf)
 - [TCG TPM Library part 3: Commands, section 11.1](https://trustedcomputinggroup.org/wp-content/uploads/TCG_TPM2_r1p59_Part3_Commands_pub.pdf)