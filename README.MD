# uOSCORE / uEDHOC 

This repository contains C implementations  for constrained (and non-constrained) devices of the IETF protocols:

* [OSOCRE (RFC8613)](https://tools.ietf.org/html/rfc8613) and
* [EDHOC (draft-ietf-lake-edhoc-15 )](https://datatracker.ietf.org/doc/html/draft-ietf-lake-edhoc-15). 

Main features of uOSCORE and uEDHOC are their independence from the OS, cryptographic engine and in the case of uEDHOC transport protocol. Additionally,  uOSCORE and uEDHOC use only stack memory (no heap).

For more background and evaluation in terms of speed, RAM and flash requirements see our paper [The Cost of OSCORE and EDHOC for Constrained Devices](https://arxiv.org/pdf/2103.13832.pdf).

## How to Build and Link

* check the configurations in `makefile_config.mk` and adjust them if necessary
* run `make` 
* link the static library `build/libuoscore-uedhoc.a` in your project

## Project (Folder) Structure

```
.
|---cddl_models/
|---externals/
|---inc/
|---samples/
|---scripts/
|---src/
|---test/
|---test_vectors/
```
* The folder `cddl_models` contains CDDL models for all CBOR structures.
* The folder `externals` contains the external libraries and tools as git submodules.
* The folder `inc` contains all header file.
* The folder `samples` contains some usage examples.
* The folder `scripts` contains scripts for generatinc C code from CDDL models and converting the json formatted EDHOC test vectors to a C header
* The folder `src` contains all source file.
* The folder `test` contains automated tests. 
* The folder `test_vectors` contains tests vectors. 

## API and Usage Model

#### uOSCORE

The API of uOSCORE consists of three functions: 
* `oscore_context_init()`,
*  `coap2oscore()` and 
*  `oscore2coap()`.

`coap2oscore()` and `oscore2coap()` convert CoAP to OSCORE packets and vice versa. `oscore_context_init()` initializes the OSCORE security context. 

First, `oscore_context_init()` function needs to be called on the client and server side, then `coap2oscore()` and `oscore2coap()`  are called just before sending or receiving packets over the network.

<img src="oscore_usage.svg" alt="drawing" width="600"/>


#### uEDHOC

The API of uEDHOC consists of four functions:
*  `ephemeral_dh_key_gen()`
* `edhoc_initiator_run()`,
* `edhoc_responder_run()`,
* `edhoc_exporter()`,

`ephemeral_dh_key_gen()` is used to generate fresh ephemeral DH keys before running the protocol. This function requires a random seed suable for cryptographic purposes. `edhoc_initiator_run()` and `edhoc_responder_run() ` has to be called on the initiator and responder side respectively. They return the External Authorization data `EAD_x`,  the derived shared secret `PRK_out`.   `PRK_out` is used as input for `edhoc_exporter()` to derive application specific keys, e.g., OSCORE master secret and OSCORE master salt.

The EDHOC protocol requires the exchange of three messages (and an optional message 4) which is independent of the underlying message transport protocol. For example [appendix-A.2 in the EDHOC specification](https://datatracker.ietf.org/doc/html/draft-ietf-lake-edhoc-15#appendix-A.2) describes how  EDHOC can be transferred over CoAP, however CoAP is not mandatory. In order to be independent of the transport protocol uEDHOC uses two callback functions which need to be implemented by the user for handling the sending and receiving of messages. These functions are:

```c
/**
 * @brief   The user should call inside this function its send function. 
 *
 * 
 * @param   sock a pointer used to identify the rx chanel, 
 *          e.g. a socket handler
 * @param   data pointer to the data to be send
 * @param   data_len length of the data
 */
enum err tx(void *sock, uint8_t *data, uint32_t data_len);

/**
 * @brief   The user should call inside this function its receive 
 *          function and copy the received data in the buffer <data>.
 *          The length of the buffer <data> must be 
 *          checked before copying into it by using <data_len>. 
 *          After copying the length of the received data should be written 
 *          in <data_len>.
 *
 * 
 * @param   sock a pointer used to identify the rx chanel, 
 *          e.g. a socket handler
 * @param   data pointer to a buffer where the edhoc message must be copied 
 * @param   data_len length of the received data. When this function is 
 *          called inside EDHOC <data_len> is initialized with the actual 
 *          available length of the <data>.
 */
enum err rx(void *sock, uint8_t *data, uint32_t *data_len);
```

Note that uEDHOC does not provide correlation of messages. Correlation may be handled on the transport layer completely or partially. In cases when the correlation cannot be handled by the transport protocol the edhoc message needs to be prepended with a connection identifier, that is used on the other side to determine to which session a given message belongs. In order to remain conform with the specification in the cases where the transport cannot handle correlation a connection identifier needs to be prepended in `tx()` function and removed in the `rx()` function.


## Supported Cipher Suites

##### uOSCORE

| Algorithms                  |
| --------------------------- |
| AES-CCM-16-64-128,  SHA-256 |

##### uEDHOC


| Suit | Algorithms                                                                 |
| ---- | -------------------------------------------------------------------------- |
| 0    | AES-CCM-16-64-128, SHA-256, 8, X25519, EdDSA, AES-CCM-16-64-128, SHA-256   |
| 1    | AES-CCM-16-128-128, SHA-256, 16, X25519, EdDSA, AES-CCM-16-64-128, SHA-256 |
| 2    | AES-CCM-16-64-128, SHA-256, 8, P-256, ES256, AES-CCM-16-64-128, SHA-256    |
| 3    | AES-CCM-16-128-128, SHA-256, 16, P-256, ES256, AES-CCM-16-64-128, SHA-256  |




## Using Different Cryptographic Libraries or Hardware Accelerators

The logic of uOSCORE and uEDHOC is independent form the cryptographic library, i.e., the cryptographic library can easily be exchanged by the user. For that the user needs to provide implementations for the functions specified in `crypto_wrapper.c`. 

## Additional configuration options
Following preprocessor directives allow for better memory usage adjustments:

* **edhoc.h**

   * EDHOC_BUF_SIZES_RPK - select when authentication can be based on raw public keys, or
   * EDHOC_BUF_SIZES_C509_CERT - select when authentication can be based on CBOR encoded certificates, or
   * EDHOC_BUF_SIZES_X509_CERT - select when authentication can be based on X.509 certificates

*  **oscore.h**

   * OSCORE_MAX_PLAINTEXT_LEN - expected maximal length of OSCORE packet
   * E_OPTIONS_BUFF_MAX_LEN - expected maximal length of buffer with all encrypted CoAP options
   * I_OPTIONS_BUFF_MAX_LEN - expected maximal length of buffer with all not encrypted CoAP options
