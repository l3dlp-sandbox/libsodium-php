[![Build Status](https://travis-ci.org/jedisct1/libsodium-php.svg?branch=master)](https://travis-ci.org/jedisct1/libsodium-php?branch=master)

libsodium-php
=============

A simple, low-level PHP extension for [libsodium](https://github.com/jedisct1/libsodium).

Requires libsodium >= 1.0.9 and PHP >= 7.0.0.

Full documentation here:
[Using Libsodium in PHP Projects](https://paragonie.com/book/pecl-libsodium),
a guide to using the libsodium PHP extension for modern, secure, and
fast cryptography.

libsodium-php 1.x vs libsodium-php 2.x
======================================

This extension was originally named `libsodium`. The module was named
`libsodium.so` or `libsodium.dll`.

All the related functions and constants were contained within the
`\Sodium\` namespace.

This extension was accepted to be distributed with PHP >= 7.2, albeit
with a couple breaking changes:

- No more `\Sodium\` namespace. Everything must be in the global
namespace.
- The extension should be renamed `sodium`. So, the module becomes
`sodium.so` or `sodium.dll`.

The standalone extension (this repository; also the extension
available on PECL) was updated to match these expectations, so that
its API is compatible with what will be shipped with PHP 7.2.

libsodium-php 2.x is thus not compatible with libsodium-php 1.x.

The 1.x branch will not receive any public updates any more.

libsodium-php 1.x compatibility API for libsodium-php 2.x
==========================================================

Fortunately, for projects using the 1.x API, or willing to use it, a
compatibility layer is available.

[Polyfill Libsodium](https://github.com/mollie/polyfill-libsodium)
brings the `\Sodium\` namespace back.

Examples
========

## Encrypt a single message using a secret key

Encryption:

```php
$secret_key = sodium_crypto_secretbox_keygen();
$message = 'Sensitive information';

$nonce = random_bytes(SODIUM_CRYPTO_SECRETBOX_NONCEBYTES);
$encrypted_message = sodium_crypto_secretbox($message,
                                             $nonce, $secret_key);
```

Decryption:

```php
$decrypted_message = sodium_crypto_secretbox_open($encrypted_message,
                                                  $nonce, $secret_key);
```

How it works:

`$secret_key` is a secret key. Not a password. It's binary data, not
something designed to be human readable, but rather to have a key
space as large as possible for a given length.
The `keygen()` function creates such a key. That has to remain secret,
as it is used both to encrypt and decrypt data.

`$nonce` is a unique value. Like the secret, its length is fixed. But
it doesn't have to be secret, and can be sent along with the encrypted
message. The nonce doesn't have to be unpredicable either. It just has
to be unique for a given key. With the `secretbox()` API, using
`random_bytes()` is a totally fine way to generate nonces.

Encrypted messages are slightly larger than unencrypted messages,
because they include an authenticator, used by the decryption function
to check that the content was not altered.

## Encrypt a file using a secret key

```php
$secret_key = sodium_crypto_secretstream_xchacha20poly1305_keygen();
$input_file = '/tmp/example.original';
$encrypted_file = '/tmp/example.enc';
$chunk_size = 4096;

$fd_in = fopen($input_file, 'rb');
$fd_out = fopen($encrypted_file, 'wb');

list($stream, $header) = sodium_crypto_secretstream_xchacha20poly1305_init_push($secret_key);

fwrite($fd_out, $header);

$tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_MESSAGE;
do {
    $chunk = fread($fd_in, $chunk_size);
    if (stream_get_meta_data($fd_in)['unread_bytes'] <= 0) {
        $tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL;
    }
    $encrypted_chunk = sodium_crypto_secretstream_xchacha20poly1305_push($stream, $chunk, '', $tag);
    fwrite($fd_out, $encrypted_chunk);
} while ($tag !== SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL);

fclose($fd_out);
fclose($fd_in);
```

Decrypt the file:

```php
$decrypted_file = '/tmp/example.dec';

$fd_in = fopen($encrypted_file, 'rb');
$fd_out = fopen($decrypted_file, 'wb');

$header = fread($fd_in, SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_HEADERBYTES);

$stream = sodium_crypto_secretstream_xchacha20poly1305_init_pull($header, $secret_key);

$tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_MESSAGE;
while (stream_get_meta_data($fd_in)['unread_bytes'] > 0 &&
       $tag !== SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL) {
    $chunk = fread($fd_in, $chunk_size + SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_ABYTES);
    list($decrypted_chunk, $tag) = sodium_crypto_secretstream_xchacha20poly1305_pull($stream, $chunk);
    fwrite($fd_out, $decrypted_chunk);
}
$ok = stream_get_meta_data($fd_in)['unread_bytes'] <= 0;

fclose($fd_out);
fclose($fd_in);

if (!$ok) {
    die('Invalid/corrupted input');
}
```

How it works:

There's a little bit more code than in the previous example.

In fact, `crypto_secretbox()` would work to encrypt as file, but only
if that file is pretty small. Since we have to provide the entire
content as a string, it has to fit in memory.

If the file is large, we can split it into small chunks, and encrypt
chunks individually.

By doing do, we can encrypt arbitrary large files. But we need to make
sure that chunks cannot be deleted, truncated, duplicated and
reordered. In other words, we don't have a single "message", but a
stream of messages, and during the decryption process, we need a way
to check that the whole stream matches what we encrypted.

## Encrypt a file using a key derived from a password:

```php
$password = 'password';
$input_file = '/tmp/example.original';
$encrypted_file = '/tmp/example.enc';
$chunk_size = 4096;

$alg = SODIUM_CRYPTO_PWHASH_ALG_DEFAULT;
$opslimit = SODIUM_CRYPTO_PWHASH_OPSLIMIT_MODERATE;
$memlimit = SODIUM_CRYPTO_PWHASH_MEMLIMIT_MODERATE;
$salt = random_bytes(SODIUM_CRYPTO_PWHASH_SALTBYTES);

$secret_key = sodium_crypto_pwhash(SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_KEYBYTES,
                                   $password, $salt, $opslimit, $memlimit, $alg);

$fd_in = fopen($input_file, 'rb');
$fd_out = fopen($encrypted_file, 'wb');

fwrite($fd_out, pack('C', $alg));
fwrite($fd_out, pack('P', $opslimit));
fwrite($fd_out, pack('P', $memlimit));
fwrite($fd_out, $salt);

list($stream, $header) = sodium_crypto_secretstream_xchacha20poly1305_init_push($secret_key);

fwrite($fd_out, $header);

$tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_MESSAGE;
do {
    $chunk = fread($fd_in, $chunk_size);
    if (stream_get_meta_data($fd_in)['unread_bytes'] <= 0) {
        $tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL;
    }
    $encrypted_chunk = sodium_crypto_secretstream_xchacha20poly1305_push($stream, $chunk, '', $tag);
    fwrite($fd_out, $encrypted_chunk);
} while ($tag !== SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL);

fclose($fd_out);
fclose($fd_in);
```

Read the stored parameters and decrypt the file:

```php
$decrypted_file = '/tmp/example.dec';

$fd_in = fopen($encrypted_file, 'rb');
$fd_out = fopen($decrypted_file, 'wb');

$alg = unpack('C', fread($fd_in, 1))[1];
$opslimit = unpack('P', fread($fd_in, 8))[1];
$memlimit = unpack('P', fread($fd_in, 8))[1];
$salt = fread($fd_in, SODIUM_CRYPTO_PWHASH_SALTBYTES);

$header = fread($fd_in, SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_HEADERBYTES);

$secret_key = sodium_crypto_pwhash(SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_KEYBYTES,
                                   $password, $salt, $opslimit, $memlimit, $alg);

$stream = sodium_crypto_secretstream_xchacha20poly1305_init_pull($header, $secret_key);

$tag = SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_MESSAGE;
while (stream_get_meta_data($fd_in)['unread_bytes'] > 0 &&
       $tag !== SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_TAG_FINAL) {
    $chunk = fread($fd_in, $chunk_size + SODIUM_CRYPTO_SECRETSTREAM_XCHACHA20POLY1305_ABYTES);
    list($decrypted_chunk, $tag) = sodium_crypto_secretstream_xchacha20poly1305_pull($stream, $chunk);
    fwrite($fd_out, $decrypted_chunk);
}
$ok = stream_get_meta_data($fd_in)['unread_bytes'] <= 0;

fclose($fd_out);
fclose($fd_in);

if (!$ok) {
    die('Invalid/corrupted input');
}
```
