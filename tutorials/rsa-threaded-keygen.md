Threaded RSA Key Generation and Decryption
==========================================
CowCrypt supports RSA encryption / decryption, as well as a [FIPS 186-4][1]
compliant secure key generator. RSA cryptography involves mathematics with very
large prime numbers. Depending on the browser and CPU, these operations can be
resource-intensive (slow). Because JavaScript is typically single-threaded,
executing these operations synchronously can cause the browser to become
unresponsive while it waits for the functions to complete. This can lead to an
apparently hung browser, or the dreaded "a script on this page is taking too
long" warning box.

Fortunately, many modern browsers support the [Web Workers][2] API, a new part
of the HTML 5 spec. This allows us to execute resource-intensive operations in
a separate thread, which runs asynchronously and "phones home" to the main
browser thread when the work is complete. Using Web Workers, CowCrypt can do
expensive RSA operations in the background without impacting perceived browser
performance or responsiveness.


Table of Contents
-----------------
* [Requirements][5]
* [Generating an RSA Key][6]
	- [Generate public exponent _e_][7]
	- [A note on BigInt values][8]
	- [Generate random primes _p_ and _q_][9]
	- [Compute the private key from _p_ and _q_][10]
* [Threaded RSA Decryption][11]
* [**crypto_math.js Threaded Messaging Reference**][12]
	- [Messages sent to the worker][13]
	- [Messages sent from the worker][14]


Requirements
------------
* Browser support for the [Web Workers][2] API
* Browser support for _[window.crypto.getRandomValues][3]_ (for key generation)
* CowCrypt's crypto_math.js (for key generation), plus rsa.js and convert.js
  (for encryption / decryption)


Generating an RSA key
---------------------
An RSA key is built up from several values. We'll generate all of these:

* _e_:		The public exponent
* _p_:		First private prime factor
* _q_:		Second private prime factor
* _n_:		The public modulus, the product of p × q. (The security of RSA
			depends on it being difficult to factor p and q out of this number)
* _Φ(n)_:	(phi of n) The product of (p - 1) × (q - 1). Used to compute _d_
* _u_:		Multiplicative inverse of the lesser(p, q) mod greater(p, q)
* _d_:		The private exponent


#### Generate the public exponent _e_

First, we generate our public exponent _e_. This can be any odd integer greater
than 2^16 and less than 2^256. It doesn't have to be random (65537 will do just
fine), but if you want a random value, CowCrypt has you covered:

```javascript

	// Generate random public exponent
    var e = crypto_math.get_random_public_exponent();
```

This returns a BigInt value (not an integer). If you're wondering what the hell
that is, read on.


#### A note on BigInt values

RSA uses huge integers, sometimes upwards of 2048 bits. JavaScript's native
support for integers tops out at 53 bits, so we're using a slightly modified
version of [Leemon's excellent BigInt.js][4] library to give us those extra
bits. Leemon's code is super fast, but it's kind of ugly, so we've wrapped it
up inside crypto_math.js to make it feel a bit more object-oriented.

CowCrypt's RSA functions expect BigInt values. If you have a big number, say,
812345834502348952793, and you want to convert it to a BigInt, you can parse it
from a string of digits:

```javascript

    var big_int = new BigInt().parse("812345834502348952793");
```

Keep in mind: if you're going to be clever and choose your own public exponent
_e_, you'll need to parse it as a BigInt, or else EVERYTHING WILL FAIL.


#### Generating the large primes _p_ and _q_

CowCrypt's RSA key generation supports 2048- and 3072-bit key sizes. These keys
are constructed using the product of two large prime numbers _p_ and _q_, which
must be 1024- or 1536-bit BigInts (respective to the key sizes). CowCrypt uses
an algorithm based on the Miller-Rabin probabilistic primality test defined in
[FIPS 186-4][1]. This generates numbers which are _probably_ prime, within a
miniscule margin of error (less than 2 to the negative 80th power). While other
algorithms exist that generate _provably_ prime numbers, they are much slower,
and as we'll see, the Miller-Rabin method is already slow enough. In any case,
it's secure enough for US government use.

Depending on the hardware used, these primes can take between a few seconds to
over a minute to generate. We're going to use the Web Workers API to do this in
a separate thread, so as not to freeze up the browser. To set this up, we'll
define a function to create the Worker thread and listen for events from it!!

```javascript

	/**
	 *	Creates a Web Worker to generate a probable prime of a specified
	 *	security length. Executes a callback upon completion.
	 *
	 *	@param {BigInt} e			The pre-selected public exponent
	 *	@param {Number} nlen		(2048 or 3072) The desired security length
	 *	@param {function} callback	Callback function to execute on completion
	 *	@param {BigInt} [p]			OPTIONAL value for p. If this is passed in,
	 *								then we're obviously generating q (and some
	 *								additional constraints will be used)
	 */
	var generate_prime_threaded = function(e, nlen, callback, p)
	{
		// If no value for p is passed in, default it to false
		p || (p = false);

		// Define the Web Worker
		var worker = new Worker('crypto_math.js');

		// Listen for event messages passed back from the worker
		worker.addEventListener('message', function(e) {
			var data = e.data;

			switch (data.cmd)
			{
				// we can only get "secure" random numbers from the main thread
				case 'get_csprng_random_values':
					var _bitlen = data.request.bits;
					var _rand = crypto_math.get_csprng_random_values(_bitlen);
					worker.postMessage({
						cmd: 'put_csprng_random_values',
						response: {
							random_values: _rand
						}
					});
					break;

				// handle an error in the generation
				case 'put_error':
					console.log('OMG AN ERROR HAS OCCURRED! ', data.error.msg);
					worker.terminate();
					break;

				// log data to the console (for debugging purposes)
				case 'put_console_log':
					console.log(data.response.msg);
					break;

				// GOT OUR PROBABLE PRIME NUMBER! Callback and kill the thread
				case 'put_probable_prime':
					callback(data.response.prime);
					worker.terminate();
					break;
			}
		}, false);

		// Send the get_probable_prime command to start the process
		worker.postMessage({
			cmd: 'get_probable_prime',
			request: {
				e: e,
				nlen: nlen,
				p: p		// either a BigInt or false
			}
		});
	}
```

Let's pick this apart starting from the top. First, notice that this function
takes an optional input parameter p. We'll use this same function to generate
both p and q. So when we pass p into it, then the Worker knows that it should
generate q, which has some additional constraints on it (based on the value p).

Next, the worker is defined and an event listener is attached that performs
specific actions when the worker sends a "message" event to the main thread.
See below for a full list of the messages that can be sent to or from the
worker, but most of these are self-explanatory: "put_error" throws an error.
"put_console_log" writes debug data to the JavaScript console.
"put_probable_prime" is called upon completion, and it kills the worker thread
and executes the callback function.

The only real oddball message is "get_csprng_random_values". This is sent when
the worker needs some cryptographically-secure random numbers (it turns out
that being able to get random numbers is pretty important when you're looking
for random prime numbers). When the main thread gets this message, it gets some
random numbers from the crypto_math.get_csprng_random_values method, and then
sends them back to the worker via the "put_csprng_random_values" message. You
might wonder why the worker doesn't just get the random numbers on its own
without bothering the main thread. Well it's because
crypto_math.get_csprng_random_values depends on window.crypto.getRandomValues,
and worker threads have no access to the window object. So the worker has to
phone home to the main thread and ask the main thread to get its random values
for it. Ugly, but effective.

After the event listener is defined, a message is sent to the worker with the
"get_probable_prime" command, telling it to start the generation process.

Next, we wire up a series of callback functions to handle our random primes
p and q, and ultimately generate the private key:

```javascript

	// define our desired security length (either 2048 or 3072)
	var nlen = 2048;

	// define our private key variables
	var p, q, n, phi_n, d, u;

	// callback function after p is generated
	var generate_p_complete = function(prime)
	{
		p = prime;

		// call the prime generator again to generate q (passing in p)
		generate_prime_threaded(e, nlen, generate_q_complete, p);
	};

	// callback function after q is generated, completes the key generation
	var generate_q_complete = function(prime)
	{
		q = prime;

		// compute the rest of the private key using e, p and q
		var inverse_data = crypto_math.compute_rsa_key_inverse_data(e, p, q);

		n		= inverse_data.n;
		phi_n	= inverse_data.phi_n;
		d		= inverse_data.d;
		u		= inverse_data.u;

		// The order of p and q may have been swapped, such that p < q
		p		= inverse_data.p;
		q		= inverse_data.q;

		// Pass your values into a CowCrypt RSAKey object! lolz
		var key	= new RSAKey({e: e, n: n, d: d, p: p, q: q, u: u});

		// ALL DONE! LOL!
	};

	// call our prime generator, callback to generate_p_complete on completion
	generate_prime_threaded(e, nlen, generate_p_complete);
```

This is pretty easy. We first define our callback functions for primes p and q.
Then we call generate_prime_threaded, passing in the generate_p_complete
callback, and away it goes. The process ends up looking like this:

generate_prime_threaded => generate_p_complete => generate_prime_threaded(p) =>
generate_q_complete

#### Compute the private key from _p_ and _q_

Once we have primes p and q, we call crypto_math.compute_rsa_key_inverse_data
to generate the rest of the private key values, and that's it! You now have a
full RSA public/private key pair and you're ready to encrypt and decrypt data!!


Threaded RSA Decryption
-----------------------
We don't have to worry about using a Worker thread for encryption, because the
public exponent is usually relatively small. But decryption can be pretty slow.
Fortunately this is a bit more straightforward than key generation!

In this example, we'll assume that the encrypted data is base64-encoded.

```javascript

	var ciphertext = '{YOUR BASE64-ENCODED RSA ENCRYPTED STRING HERE}'

	var worker = new Worker('/library/crypto_math.js');

	worker.addEventListener('message', function(e) {
		var data = e.data;

		switch (data.cmd) {
			case 'put_rsa_decrypt':
				decryption_complete_callback(data.response.decrypted)
				worker.terminate();
				break;
		}
	}, false);

	worker.postMessage({
		cmd: 'get_rsa_decrypt',
		request: {
			ciphertext: convert.base64.decode(ciphertext),
			n: n,	// must be a BigInt
			d: d	// must be a BigInt
		}
	});

	var decryption_complete_callback = function(decrypted)
	{
		// Manually undo the PKCS1 v1.5 padding
		decrypted = new PKCS1_v1_5().decode(decrypted);

		// WE'RE DONE! LOLOL!
	}
```

In the above code, we define our worker, add an event listener that calls the
decryption_complete_callback function when a "put_rsa_decrypt" message is
received. We send the worker the "get_rsa_decrypt" command message to start the
decryption process, and define the decryption_complete_callback function,
which un-pads the decrypted plaintext at the end of the process. Simple enough!


crypto_math.js Worker Messaging Reference
-----------------------------------------
Due to the threaded nature of Web Workers, we can't explicitly call functions
between a worker and the main browser thread. Instead, the crypto_math.js
worker listens for and handles "message" events, and the main thread (your
JavaScript code) must do the same. Refer to the examples above to see how these
listeners are defined in the main thread. This section defines all of the
messages that can be sent to a worker thread, as well as the messages a worker
thread can send to the main thread.


#### Messages sent _to_ the crypto_math.js worker


**get_probable_prime**:
This message kicks off the Miller-Rabin random prime generation process.
The following properties must be passed into the request object:

* e:		(BigInt) Required public exponent e
* nlen:		(Number, either 2048 or 3072) The desired security bit length
* [p]:		(Optional BigInt) The random prime p (used to constrain q)


**get_rsa_encrypt**:
This message tells the worker to encrypt RSA plaintext given a public key.
The following properties must be passed into the request object:

* plaintext:	(ASCII-encoded binary string) The plaintext to encrypt
* n:			(BigInt) The modulus n
* e:			(BigInt) The public exponent e


**get_rsa_decrypt**:
This message tells the worker to decrypt RSA ciphertext given a private key.
The following properties must be passed into the request object:

* ciphertext:	(ASCII-encoded binary string) The ciphertext to decrypt
* n:			(BigInt) The modulus n
* d:			(BigInt) The private exponent d


**put_csprng_random_values**:
This returns random values from the crypto_math.get_csprng_random_values
method to the worker. The worker will request these from the main thread via
the "get_csprng_random_values" message (see below).
The following properties must be passed into the response object:

* random_values:   (Uint32Array) Result of crypto_math.get_csprng_random_values
  for the bit length specified in the "get_csprng_random_values" method.


#### Messages sent _from_ the crypto_math.js worker


**put_probable_prime**:
This message contains the probable prime generated by the worker after it
receives the "get_probable_prime" message from the main thread.
The following properties are passed in the response object:

* prime:	(BigInt) A probable prime


**put_rsa_decrypt**:
This message contains the decrypted plaintext computed after the worker gets
the "get_rsa_decrypt" mssage from the main thread.
The following properties are passed in the response object:

* plaintext:	(ASCII-encoded binary string) Decrypted plaintext. This will
  most likely still have PKCS1 v1.5 padding, so it's up to you to un-pad!


**put_console_log**:
This is used for debugging. Since the worker can't write directly to the
JavaScript console, it sends this to ask the main thread to do it instead.
The following properties are passed in the response object:

* msg:	(String) the message you will hopefully be kind enough to console.log


**put_error**:
Sent when an error occurs. It's up to the main thread to listen for this and
handle the errors. The following properties are passed in the response object:

* msg:	(String) The error message
* code: (Number) An integer error message

The following error codes are currently defined:

* 1:	Unsupported nlen security length (must be 2048 or 3072)
* 2:	Invalid public exponent e
* 3:	Probable prime generation failure (giving up after too many tries)


**get_csprng_random_values**:
The worker thread sends this message when it needs cryptographically-secure
random values. Since only the main thread can generate these (due to reliance
on the window object), the main thread must call
crypto_math.get_csprng_random_values, passing in the requested bit length,
and return the result to the worker in a "put_csprng_random_values" message.
The following properties are passed in the request object:

* bits:	(Number) the requested number of randomly generated bits


[1]: http://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.186-4.pdf
[2]: http://www.html5rocks.com/en/tutorials/workers/basics/
[3]: https://developer.mozilla.org/en-US/docs/Web/API/window.crypto.getRandomValues
[4]: http://www.leemon.com/crypto/BigInt.html
[5]: #requirements
[6]: #generating-an-rsa-key
[7]: #generate-the-public-exponent-e
[8]: #a-note-on-bigint-values
[9]: #generating-the-large-primes-p-and-q
[10]: #compute-the-private-key-from-p-and-q
[11]: #threaded-rsa-decryption
[12]: #crypto-math-js-worker-messaging-reference
[13]: #messages-sent-to-the-crypto-math-js-worker
[14]: #messages-sent-from-the-crypto-math-js-worker