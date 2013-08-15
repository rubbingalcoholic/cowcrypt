CowCrypt -- MooTools encryption libs
====================================
#### SHA-1, SHA-256, MD5, AES, CAST5, Twofish, HMAC, PBKDF2 and more!

Introduction
------------
CowCrypt provides popular hashing and encryption algorithms for the [MooTools
JavaScript framework][1]. My goal is to provide much of the same robust
functionality as the [crypto-js][2] library, but with the elegance and familiar
syntax of MooTools. The code is licensed under [the MIT License][3].

This is an early stage project and the algorithms are not fully optimized for
speed. However, many of them perform neck-in-neck with other popular crypto
libraries, and some of them are much faster (some of them are slower :p).

Please contact me if you have any questions, issues, or feature requests. Feel
free to use the GitHub Issue tracker or email me <<jeff@rubbingalcoholic.com>>

* [**Read the API documentation**][4]
* [**View demos**][5]


Quick-Start Table of Contents
-----------------------------
* [**WARNING**: Encode your strings as ASCII!][6]
* [The Hashers: MD5, SHA-1, SHA-256][7]
	- [Basic Usage][8]
	- [Streaming / Progressive Hashing Mode][9]
	- [Hasher Output Formats][10]
* [HMAC][11]
	- [Basic Usage][12]
	- [Streaming / Progressive HMAC Mode][13]
* [Symmetric Encryption: AES, CAST5, Twofish][14]
	- [Basic Usage][15]
	- [Custom Key / Initial Vector][16]
	- [Cipher Block Modes][17]
	- [Byte Padding Modes][18]
	- [Cipher Output Format][19]
	- [OpenSSL Interoperability Mode][20]
* [Asymmetric Encryption: RSA][23]
    - [A note on BigInt values and _crypto_math.js_][30]
    - [Key Generation][31]
    - [Handling RSA Keys][32]
    - [RSA Encryption][33]
    - [RSA Decryption][34]
    - [EME PKCS1 v1.5 Encoding / Decoding][35]
* [PBKDF2][21]
* [Contributing][22]


WARNING: Use ASCII strings!
---------------------------
CowCrypt's hashing and encryption functions accept string inputs, but they must
be ASCII-encoded (ie. containing no UTF8 / multibyte character code values).
The algorithms do not slow down to check whether your strings are actually
ASCII. This _will_ lead to broken or corrupted results if you pass multibyte
characters. NOTABUG! WONTFIX!

CowCrypt provides convenient conversion methods to encode UTF strings to ASCII,
but you are responsible for doing this as needed outside of the encryption and
hashing functions. Be careful!

```javascript

    var mystring	= 'ʟʘȽ ƮɦɨƧ ȘƬɌȋɴɢ ɪŜ ƜǡƇƘŸǃ ßƐŧŧëƦ ȄƞĈøÐe ĩť ƒĭƦŚȚ.';
    var mystring	= convert.utf8.encode(mystring);
    var hashed		= new SHA1().hash(mystring);
```

The Hashers
-----------
CowCrypt includes the following classes for hashing data:

* __MD5__: An old hashing algorithm that has fallen out of favor due to its
susceptibility to collision attacks. A lot of apps still use it, so here it is.

* __SHA1__: Designed to replace MD5. SHA-1 has some theoretical security
weaknesses, but it's still the standard and the most widely used hasher.

* __SHA256__: Also known as SHA-2. It was designed to address the security
problems with SHA-1. Since there's no (known) real world attack on SHA-1, it
isn't widely used.

#### Basic Hasher Usage

All of these classes extend the Hasher base class, which provides common
functionality and makes the usage syntax more or less identical. You can read
about this in the API docs. OK, here are the usage examples:

```javascript

    var hash = new MD5().hash('data data data ');
    // 760c2be99e98ae3027ae4d4c2816d6ea

    var hash = new SHA1().hash('data data data ');
    // 65bd90d5e213e8d03e87b5be5eeda3bc81faa772

    var hash = new SHA256().hash('data data data ');
    // 6b6d03945132109b4e8c035318219e9553f9e772dbf9392094492b2ea8a4e9ad
```
#### Progressive Hashing Mode

Normally you pass your data into the hash function and you're done with it. But
if you're hashing huge amounts of data, perhaps a Linux ISO, it's a big
performance drain to pass all of it around in memory at once. Since the hasher
functions really only operate on 64 bytes of data at a time, there's no reason
we should have to pass all that data into them in one call.

Using streaming (or progressive hashing) mode, we can stream data into the
hasher over multiple calls, potentially reducing our overall memory footprint.
See below.

```javascript

    var _sha1 = new SHA1();
    for (var i = 0; i < 3; i++)
        _sha1.hash('data ', {stream: true});

    var hash = _sha1.finalize();
    // 65bd90d5e213e8d03e87b5be5eeda3bc81faa772
```
#### Hasher Output Formats

By default, each of the hashers returns a hexadecimal string. You can change
this to an ASCII-encoded binary string, or an array of 32-bit integer "words"
using the return_format option for the hash and finalize methods.

```javascript

    // in normal hashing mode, pass return_format option on hash
    var hash = new SHA1().hash('data', {return_format: 'words'});

    // in streaming mode, pass return_format option into finalize
    var _sha1 = new SHA1();
    for (var i = 0; i < 3; i++)
        _sha1.hash('data ', {stream: true});

    var hash = _sha1.finalize({return_format: 'binary');
```

HMAC
----
HMAC is a method of hashing data that uses a passphrase in the hashing process.
This allows you to verify both data integrity and authenticity.

#### Basic HMAC Usage

```javascript

    var _hmac = new HMAC({
        passphrase: 'password1234lol',
        hasher: SHA1
    });
    _hmac.hash('my data to hash');
```

#### Progressive HMAC Hashing Mode

This wraps the streaming mode functionality for the Hasher subclass instance
used by HMAC. [See above][9] for a more complete explanation.

```javascript

    var _hmac = new HMAC({ passphrase: '12345678'});
    _hmac.hash('Message Part 1', {stream: true});
    _hmac.hash('Message Part 2', {stream: true});
    _hmac.hash('Message Part 3', {stream: true});
    var output = _hmac.finalize();
```

Symmetric Encryption
--------------------
So far, CowCrypt has the following classes for symmetric encryption:

* __AES__: Also known as Rijndael, AES is effectively the world
standard symmetric block cipher. It accepts key sizes of 16, 24, and 32 bytes,
so you can choose between speed and moar security.

* __CAST5__: This is kind of an obscure algorithm. It's used internally by
OpenPGP, so it's good to have around if you were thinking about building a
JavaScript OpenPGP application.

* __Twofish__: This went head-to-head against Rijndael as a finalist in NIST's
AES-selection process. Although it lost the competition, it is a very secure
algorithm and the dude who built it (Bruce Schneier) it is a total genius.

#### Basic Symmetric Encryption Syntax

All of these classes extend the BlockCipher base class, which provides common
functionality and makes the usage syntax more or less identical. You can read
about this in the API docs.

```javascript

    // This encrypts by deriving a key from the passphrase 'mypassword1234'
    var _aes		= new AES({passhprase: 'mypassword1234'});
    var data		= convert.utf8.encode('i like ŞĩŁĿŶ ǙƝȈʗʘɗε characters');
    var encrypted	= _aes.encrypt(data);
```

#### Custom Key / Initial Vector

You can also directly specify a key and initial vector (IV).

```javascript

    var my_key	= convert.hex_to_binstring('0123456712345678234567893456789a');
    var my_iv	= convert.hex_to_binstring('9876543210fedcba');
    var _cast5	= new CAST5({key: my_key, iv: my_iv});
    var crypted	= _cast5.encrypt('ASCII string! no UTF encoding needed! lol');
```

#### Cipher Block Modes

CowCrypt includes the following block operation mode classes:

* __CBC__ (default)
* __CFB__
* __ECB__ (weak sauce)

To customize, initialize your cipher with the block_mode option set to a
reference of your preferred block operation mode class.

```javascript

    // initialize with ECB mode:
    var cipher	= new Twofish({passphrase: '1234', block_mode: ECB});

    // initialize with CFB mode:
    var cipher	= new Twofish({passphrase: '1234', block_mode: CFB});
```

#### Byte Padding Modes

CowCrypt includes the following padding mode classes:

* __PKCS7__ (default)
* __ANSIX923__
* __ZeroPadding__

When your input plaintext doesn't match the ciphers block size, it is padded
before encryption according to whatever byte padding mode class is selected.
Initialize the cipher with the pad_mode option to override the default. Padding
will automatically be removed after decryption, except in the case of Zero
Padding, which can't be safely removed without potentially corrupting the data.

```javascript

    // initialize with Zero Padding mode:
    var cipher	= new CAST5({passphrase: '1234', pad_mode: ZeroPadding});

    // initialize with ANSIX923 mode:
    var cipher	= new Twofish({passphrase: '1234', pad_mode: ANSIX923});
```

#### Cipher Output Format

All of the BlockCipher subclasses output ASCII-encoded binary strings for both
encryption and decryption. You can easily convert this to other formats using
the provided convert library:

````javascript

    var encrypted = new AES({passphrase: '1234'}).encrypt('lol i have herpes');

    var hex = convert.binstring_to_hex(encrypted);	// convert to hex

    var b64 = convert.base64.encode(encrypted);		// convert to Base64
````

#### OpenSSL Interoperability Mode

OpenSSL uses a special format for its encrypted data which prepends salt
information to the ciphertext. CowCrypt offers an OpenSSL compatibility mode
for all BlockCipher subclasses. This changes the ciphertext output format and
passhprase-based key generation method to allow interoperability with OpenSSL.

Suppose we save the text _"this is my encrypted file, hopefully."_ into a file
called in.txt. Then we can use OpenSSL to encrypt with the shell command:

```shell
    openssl enc -aes-256-cbc -in in.txt -pass pass:"Secret Passphrase" -base64
```

The output of this command will be some Base64-encoded encrypted data stream
using a random salt (different every time). We can decrypt it with CowCrypt.

```javascript

    var openssl_data = convert.base64.decode('U2FsdGVkX18hkhQhtWqrwimjR4BBoZvK\
    tdd9MzDuuqU+pyHD+T2aA2/XvB6OY5bvaH7fTvIxGz7Yve01j25+LQ==');

    var _aes = new AES({
        block_mode: CBC,
        pad_mode: PKCS7,
        passphrase: 'Secret Passphrase',
        openssl_mode: true
    });

    var plaintext = _aes.decrypt(openssl_data);
    // outputs "this is my encrypted file, hopefully."
```

RSA Encryption
--------------
CowCrypt supports RSA encryption, decryption and key-generation using a
[FIPS-186-4][27]-compliant probable prime generation algorithm. These features
can be run in a separate thread using [HTML5 Web Workers][24] for smooth
performance. See the [Threaded RSA Key Generation and Decryption][25] tutorial
for more information on setting this up.

**[Check out the RSA demo page][29] to see everything in action.**

#### A note on BigInt values and _crypto_math.js_

RSA uses huge integers, sometimes upwards of 2048 bits. JavaScript's native
support for integers tops out at 53 bits, so we've wrapped up Leemon's awesome
[BigInt.js library][26] into _crypto_math.js_ to give us those extra bits. Keep
in mind:

* **1:**    _crypto_math.js_ is required for all RSA functionality
* **2:**    The values used for public and private keys are BigInts, and cannot
            be treated like normal integers. This should be transparent to you,
            assuming you use CowCrypt's built in RSA functions.

You can parse a BigInt value from a string of digits as follows:

```javascript

    var big_int = new BigInt().parse("812345834502348952793");
```

#### Key Generation

CowCrypt can generate 2048- or 3072-bit RSA keys. The process is CPU-intensive
and can take from several seconds to over a minute, depending on the hardware
used. To prevent the browser from becoming unresponsive, RSA key generation
must be done in a separate thread. This allows the algorithms to work in the
background without impacting perceived browser performance. See the
[Threaded RSA Key Generation and Decryption][25] tutorial for more information.


#### Handling RSA Keys

The non-threaded RSA encryption / decryption methods in the _RSA_ class expect
you to pass an _RSAKey_ object. This object holds the BigInt values associated
with a public or private key. Private keys have some optional properties that
store additional data about how the key was generated. This will be useful for
serializing OpenPGP key data packets once support for that is implemented ;)

```javascript

    // The threaded key generation process will generate all these input values
    var key = new RSAKey({
        n: key_modulus,             // required for all keys
        e: public_exponent,         // required for public keys
        d: private_exponent,        // required for private keys
        p: large_prime_p,           // optional for private keys
        q: large_prime_q,           // optional for private keys
        u: mult_inverse_p_mod_q     // optional for private keys
    });
```

#### RSA Encryption

CowCrypt uses the EME PKCS1 v1.5 encoding method for RSA plaintexts specified
in [RFC-4880][28] (the OpenPGP spec). This is can encrypt messages up to 11
bytes less than the byte-length of the modulus, and is useful for symmetric key
establishment (a la OpenPGP).

```javascript

    var public_key  = {YOUR RSAKey Instance Here};
    var plaintext   = "Your time is up I will no longer play for you.";
    var ciphertext  = new RSA().encrypt(plaintext, public_key);
```

#### RSA Decryption

Once you actually have a private key and some ciphertext, decrypting is easy.
Note that decrypting is generally a lot slower than encrypting, so this is a
good thing to do in [a separate Worker thread][25].

```javascript

    var private_key = {YOUR RSAKey Instance Here};

    // generally RSA ciphertext is Base64-encoded. Convert to a binary string!
    ciphertext      = convert.base64.decode(ciphertext);

    var plaintext   = new RSA().decrypt(ciphertext, private_key);
```

#### EME PKCS1 v1.5 Encoding and Decoding

If you're using the threaded RSA functions for encryption and decryption, you
must use encode your plaintext before encryption, and decode your decrypted
plaintext after decryption using the PKCS1 v1.5 class. The threaded functions
assume you've already done this, and they won't break, but your ciphertext will
be a lot less secure if you don't encode properly. Anyhoo, this is easy:

```javascript

    // BEFORE ENCRYPTING
    var encoded_plaintext = new PKCS1_v1_5().encode(plaintext);

    // ...

    // AFTER DECRYPTING
    var decoded_plaintext = new PKCS1_v1_5().decode(encoded_plaintext);
```

PBKDF2
------
PBKDF2 is a standard for turning a passphrase into a symmetric key using an
optional salt. This is used internally by the symmetric ciphers when you
initialize with a passphrase instead of explicitly passing a key and initial
vector. Note that the PBKDF2 compute method returns an ASCII-encoded binary
string, just like how the symmetric ciphers do.

```javascript

    var _kdf = new PBKDF2({
        key_size: 32,			// note the key size is in bytes
        hasher: SHA256,			// PBKDF2 uses HMAC internally
        iterations: 1000		// moar = bettar (slowar)
    });
    var key = _kdf.compute('password1234', 'arbitrary salt value');
```

Contributing
------------
Please let me know if you have any suggestions for improvements. If you're code
savvy, fork the project and make the change yourself! I will do my best to help
if something doesn't work or isn't clear. You can find me on Twitter
@rubbingalcohol


[1]: http://mootools.net
[2]: http://code.google.com/p/crypto-js
[3]: http://opensource.org/licenses/MIT
[4]: http://rubbingalcoholic.github.io/cowcrypt/api
[5]: http://rubbingalcoholic.github.io/cowcrypt/demos
[6]: #warning-use-ascii-strings
[7]: #the-hashers
[8]: #basic-hasher-usage
[9]: #progressive-hashing-mode
[10]: #hasher-output-formats
[11]: #hmac
[12]: #basic-hmac-usage
[13]: #progressive-hmac-hashing-mode
[14]: #symmetric-encryption
[15]: #basic-symmetric-encryption-syntax
[16]: #custom-key--initial-vector
[17]: #cipher-block-modes
[18]: #byte-padding-modes
[19]: #cipher-output-format
[20]: #openssl-interoperability-mode
[21]: #pbkdf2
[22]: #contributing
[23]: #rsa-encryption
[24]: http://www.html5rocks.com/en/tutorials/workers/basics/
[25]: https://github.com/rubbingalcoholic/cowcrypt/blob/master/tutorials/rsa-threaded-keygen.md
[26]: http://www.leemon.com/crypto/BigInt.html
[27]: http://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf
[28]: http://tools.ietf.org/rfc/rfc4880.txt
[29]: http://rubbingalcoholic.github.io/cowcrypt/demos/rsa.html
[30]: #a-note-on-bigInt-values-and--crypto-math-js--
[31]: #key-generation
[32]: #handling-rsa-keys
[33]: #rsa-encryption
[34]: #rsa-decryption
[35]: #eme-pkcs1-v-1-5-encoding-and-decoding