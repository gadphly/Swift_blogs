EVP = envelope 

# Public Key Algorithms

## EVP_PKEY_*

Uses a public key algorithm.
Uses EVP_PKEY_CTX

- EVP_PKEY_encrypt / decrypt
- EVP_PKEY_sign / verify: Does not hash data to be signed, so normally uses to sign digests. For normal signing, use EVP_DigestSign
- EVP_PKEY_keygen

## EVP_*

Uses EVP_CIPHER_CTX

- EVP_Sign / Verify: Older APIs for sign/verify. Used for both HMAC and RSA
- EVP_Seal / Open: RSA+AES encryption
- EVP_DigestSign / DigestVerify: New APIs for sign/verify
