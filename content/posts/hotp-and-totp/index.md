---
categories:
- Algorithms
date: 2021-01-23T11:04:43-07:00
description: In this post, I will tell you all that you need to know about how those one time passwords that you receive in texts, emails, or authentication programs work.
draft: false
math:
  enable: true
resources:
- name: "featured-image"
  src: "feature.jpg"
- name: "featured-image-preview"
  src: "feature_small.jpg"
tags:
- HOTP
- Password
- TOTP
title: "All About One Time Passwords"
---
Two factor authentication is happening everywhere now. Corporate logins and many websites are now texting, emailing, or requiring you to enter a one time passcode in order to gain access. In this post, I will show you how these codes are being generated and how you can add them to your applications.

<!--more-->

You are seeing this a lot lately. Most websites or corporate logins are starting to require more than just a username and password for authentication. You may be receiving a six or eight digit passcode via a text message, from an email, or using a tool such as 1Password or Google Authenticator to generate a temporary and time-limited passcode to prove your identity. This is called **two-factor authentication**, or **2FA**. Have you ever asked yourself where these codes come from? How do these companies generate and validate these codes? How do they work and are they secure? In this post, we'll dive into the algorithms and standards behind these passcodes to show you how they work and how you can use them with your own applications.

## A Tale of Two RFCs

The passcodes that we talk about are technically considered one time passwords. The idea is that you use the password once and never use it again. Many of these passcodes are time sensitive. They may expire in 30 seconds or 15 minutes. But there's usually a time limit involved when using the passwords for online services.

There are two [Internet RFCs](https://en.wikipedia.org/wiki/Request_for_Comments) that provide the algorithms for implementing one time passwords:

* [RFC 4226: HOTP: An HMAC-Based One-Time Password Algorithm](https://tools.ietf.org/html/rfc4226)
* [RFC 6238: TOTP: Time-Based One-Time Password Algorithm](https://tools.ietf.org/html/rfc6238)

HOTP is the base algorithm. HOTP works by using an incrementing counter. Every time a password is requested, a new password is generated based off of the new value of the counter. The same password cannot be reused more than once. TOTP is an application of HOTP and generates a counter using time. TOTP uses the concept of a time step, such as 30 seconds. The current time is matched to a time bucket with a counter value. The one time password is only valid within that same time bucket.

## A Quick Primer on Cryptography

I'm going to stop quickly to provide a quick primer on the cryptography information that you need to know to understand these algorithms. Both HOTP and TOTP are based on existing and trusted cryptographic algorithms for verifying message integrity and uniqueness. Cryptography is a big topic, so I am limiting my explanations to those concepts that you need to know to understand the HOTP and TOTP algorithms.

The first concept that I want to teach you about is what we call a **hash**. Let's say that we have a message that we call $ M $. The message can be anything. It could be a string containing your name. It could be a JSON blob. It could be a file. It can be anything that can be represented as a string of bytes or octets (8-bit values). Given a message $ M $, there are functions that we call **hash functions** that will use $ M $ to generate a unique hash value that we'll call $ H $:

$$ F(M) = H $$

The hash function $ F(M) $ is a [pure function](https://en.wikipedia.org/wiki/Pure_function). When called with the same value of $ M $, it will produce the same value $ H $. $ F(M) $ does not alter $ M $ in any way.

The family of hash functions that we will use for HOTP and TOTP are what we call cryptographic hash functions. These functions have been engineered such that the probability of ever encountering two messages $ M_1 $ and $ M_2 $ is so improbably that it is unlikely to happen. When this happens, it is called a **collision**. A collision is theoretically possible based on the size of the hash value $ H $ in bits. The SHA-1 function, for example, produces a hash value that is 160 bits in length, so there are $ 2^{160} $ possible values and an attack may need to throw that many messages at the hash function to find a collision.

Let's say that we have two users named Alice and Bob. Alice sends Bob a message. When Bob receives the message, how does Bob know that the message is correct? If the message came over the Internet, there could have been a bad router along the way that flipped a bit from 1 to 0 or 0 to 1. Then when Bob receives the message, the message he receives is corrupted and he will probably misread the message. Now let's say that Alice sends Bob both the message $ M $ and the hash value $ H $. Now if Bob and Alice both agree on the same hash function, Bob can generate a hash value from the message using $ F(M) = H' $. If $ H $ does not equal $ H` $, then Bob knows that he did not receive the message correctly. If $ H $ equals $ H' $, then Bob knows that the message was transmitted correctly.

The next problem is how does Bob know that the message came from Alice? Bob can use the hash to verify that the message is correct. But Bob cannot verify the source. Another actor, Charlie, could use the same hash function that Alice and Bob are using and send a message to Bob while pretending to be Alice. How can Bob verify that the message $ M $ was sent from Alice to Bob?

To verify the message, we have two cryptographic options:

* Message authentication codes
* Digital signatures

We're going to focus on message authentication codes in this post. A message authentication code is generated by another function $ MAC(K, H) $ that takes a hash $ H $ and a secret $ K $ and generates a new hash value $ C $. Like the hash function, $ MAC(K, H) $ is a pure function and will only generate the code $ C $ using the same secret $ K $ and hash $ H $:

$$ MAC(K, H) = C $$

Now, back to Alice and Bob, if Alice sends Bob a message $ M $ and the message authentication code $ C $ instead of the hash $ H $, and if Bob and Alice agree on a secret $ K $, Bob can authenticate that the message came from Alice and not Charlie, assuming that Charlie does not know the secret that Alice and Bob are sharing. Bob can calculate the hash $ H $ of message $ M $, and then using the secret calculate the authentication code $ C $ using the message authentication code function $ MAC(K, H) $. If the authentication codes match, then the message is authentic and can be assumed to have originated from Alice. Remember that this works only if it can be guaranteed that the secret that Alice and Bob share is kept safe and never exposed to Charlie.

## HOTP

HOTP is based on the [HMAC](https://en.wikipedia.org/wiki/HMAC) message authentication code algorithm. HMAC produces a message authentication code that is equal in length to the hash that is provided to it. HMAC also requires a shared secret value that is the same length as the hash.

HOTP works using an incrementing counter. Imagine that you are given a token that you can tap a button to generate a password. The first time that you tap the button, the internal counter is set to 0. The next time, the counter will be 1, then 2, and so on. There is no way to reset the token so that you can get the first password ever again. The secret is stored in your token and may also be stored on the server that you send your password to in order to authenticate. The server also keeps track of how many passwords you have entered, so its counter matches the token's counter.

With HOTP, we can define a function that outputs a unique password:

$$ HOTP(K, C) = P $$

$ K $ is the shared secret value. $ C $ is the current value of the incrementing counter. $ P $ is the generated password. In reality, there's two other parameters to the function that I will call $ D $:

$$ HOTP(H, K, C, D) = P $$

$ D $ is the desired length of $ P $. For example, it is common to see one time passwords that are six or eight digits in length. One time passwords are usually zero-padded on the left to ensure that the password is the desired length. $ H $ is a function of the form $ H(M) $ that is the hash function that is used to generate the one time password. For HOTP, $ C $ is a 64-bit unsigned integer value and typically starts at 0.

HOTP will begin by using the counter $ C $ to generate the hash using the hash function $ H $:

$$ H(C) = H' $$

The output $ H` $ is a sequence of bytes or octets (8 bits) that are the hash of the counter $ C $. For example, if $ H(C) $ implements the [SHA-1](https://en.wikipedia.org/wiki/SHA-1) algorithm, then $ H' $ will be 160 bits (20 bytes) in length. We then calculate the message authentication code $ A $ using the HMAC algorithm:

$$ HMAC(K, H') = A $$

The next step is to truncate $ A $ and produce the one time password. The first thing that we will do is to look at the last 4 bits of $ A $. The last 4 bits will give us an offset between 0 and 15 into $ A $ that we will use to extract out a 32-bit signed non-negative integer (it is guaranteed to be non-negative because we only use the lower 31 bits of the extracted value, so the sign bit will always be 0).

Finally, we can calculate the one time password to the specified number of digits (assume $ V $ is the 32-bit positive integer that we just extracted from the hash):

$$ P = V mod 10^D $$

The result is that we will have a code that is $ D $ digits in length or less (which we will need to zero-pad).

How does this look in code? Fortunately, this is fairly easy. I have implemented the algorithm below using Go:

{{< admonition type="warning" title="Pay Attention to Byte Order" open="true" >}}
It is important to note that HOTP assumes that binary values are stored in [big endian](https://en.wikipedia.org/wiki/Endianness) form. Most modern processors store binary values in little endian form, so it is important if you are translating this algorithm to another language or implementation to ensure that you operate on the bytes in the correct order and translate when necessary.
{{< /admonition >}}

{{< highlight go "linenos=table" >}}
package hotp

import (
    "crypto/hmac"
    "encoding/binary"
    "fmt"
    "hash"
    "math"
)

type HOTP struct {
    hash   func() hash.Hash
    secret []byte
    digits int
    format string
}

func New(hash func() hash.Hash, secret []byte, digits int) HOTP {
    format := fmt.Sprintf("%%0%dd", digits)
    return HOTP{
        hash: hash,
        secret: secret,
        digits: digits,
        format: format,
    }
}

func (h HOTP) Password(counter uint64) string {
    // Convert counter to a byte array

    counterBytes := make([]byte, 8)
    binary.BinEngian.PutUint64(counterBytes, counter)

    // Calculate the HMAC of the hash

    hmac := hmac.New(h.hash, h.secret)
    hmac.Write(counterBytes)
    hash := hmac.Sum(nil)

    // Truncate the authentication code. Get the offset from the lower 4 bits
    // of the authentication code.
    offset := hash[len(hash) - 1] & 0xf

    // Use the offset to read a 32-bit positive integer. The sign bit is
    // excluded using the mask to ensure that num is a positive integer.

    p := hash[offset:offset+4]
    num := binary.BigEndian.Uint32(p) & 0x7fffffff

    // Generate the one time passcode and zero-pad it if necessary

    d := uint64(math.Mod(float64(num), math.Pow10(h.digits)))
    return fmt.Sprintf(h.format, d)
}
{{< /highlight >}}

Appendix D of the RFC includes test outputs for the HOTP algorithm when using the string `"12345678901234567890"` as the secret value. Using the previous implementation, I should see the following:

{{< highlight go "linenos=table" >}}
secret := []byte("12345678901234567890")
hotp := hotp.New(sha1.New, secret, 6)

hotp.Password(0) // 755224
hotp.Password(1) // 287082
hotp.Password(2) // 359152
hotp.Password(3) // 969429
hotp.Password(4) // 338314
hotp.Password(5) // 254676
hotp.Password(6) // 287922
hotp.Password(7) // 162583
hotp.Password(8) // 399871
hotp.Password(9) // 520489
{{< /highlight >}}

We have succeeded in generating one time passwords!

If you have been following along, you might have identified a problem with the HOTP algorithm. What happens if our password token and the server get out of sync? What happens if the server is offline and we request a password? The next time we try to log in, the server's counter is one less than the token's. to fix this, you will need to come up with some sort of resynchronization capability between your token and server. TOTP gets around this by using time as the counter.

## TOTP

TOTP builds on top of HOTP and actually uses HOTP. But instead of using an incrementing counter, TOTP uses time, or more specifically, time buckets to generate a password that is unique and valid within a specific time period.

TOTP introduces two new values. $ T_0 $ is what we call the start time. This is a time that both parties agree to. Time for TOTP is specified as the number of seconds since $ T_0 $. The next value we'll call $ X $ and is the duration in seconds of what we will call a **time step**. Let us use 30 seconds for $ X $. This means that TOTP will generate the same password for seconds 0 through 29 of the same minute. When the number of seconds changes to 30, TOTP will give us a new password.

$ T_0 $ can be any time, but the time that is typically used is a value known as the [Unix epoch](https://en.wikipedia.org/wiki/Unix_time) which is defined as midnight UTC time on January 1, 1970. To calculate the counter value that is passed to HOTP, TOTP calculates the number of seconds between the current time and $ T_0 $, and then divides that result by the time step duration $ X $ to determine which time step is active. That time step now becomes the counter that is passed to HOTP. If the Unix epoch is not used, then $ T_0 $ is the number of seconds since midnight UTC time on January 1, 1970. Otherwise $ T_0 $ will be 0 to equal the Unix epoch.

Using $ T $ for the current time, we can define the TOTP algorithm:

$$ C = \frac{T - T_0}{X} \newline P = TOTP(H, K, C, D) $$

This can be coded easily in Go:

{{< highlight go "linenos=table" >}}
package totp

import (
    "hash"
    "time"

    "github.com/mfcollins3/hotp"
)

type TOTP struct {
    hotp      hotp.HOTP
    startTime uint64
    timestep  uint64
}

func New(
    hash func() hash.Hash,
    secret []byte,
    digits int,
    startTime time.Time,
    timeStep int
) TOTP {
    return TOTP{
        hotp:      hotp.New(hash, secret, digits),
        startTime: uint64(startTime.Unix()),
        timeStep:  uint64(timeStep)
    }
}

func (t TOTP) Password(time time.Time) string {
    unixTime := uint64(time.Unix())
    counter := (unixTime - t.startTime) / t.timeStep
    return t.hotp.Password(counter)
}
{{< /highlight >}}

I can test this out using the test values in the TOTP specification:

{{< highlight go "linenos=table" >}}
epoch := time.Unix(0, 0)

// SHA-1 (20-byte secret)

secret := []byte("12345678901234567890")
totp := totp.New(sha1.New, secret, 8, epoch, 30)

totp.Password(time.Unix(59, 0)) // 46119246
totp.Password(time.Unix(1111111109, 0)) // 68084774
totp.Password(time.Unix(1111111111, 0)) // 67062674
totp.Password(time.Unix(1234567890, 0)) // 91819424
totp.Password(time.Unix(2000000000, 0)) // 90698825
totp.Password(time.Unix(20000000000, 0)) // 77737706

// SHA-256 (32-byte secret)

secret := []byte("12345678901234567890123456789012")
totp = totp.New(sha256.New, secret, 8, epoch, 30)

totp.Password(time.Unix(59, 0)) // 46119246
totp.Password(time.Unix(1111111109, 0)) // 68084774
totp.Password(time.Unix(1111111111, 0)) // 67062674
totp.Password(time.Unix(1234567890, 0)) // 91819424
totp.Password(time.Unix(2000000000, 0)) // 90698825
totp.Password(time.Unix(20000000000, 0)) // 77737706

// SHA-512 (64-byte secret)

secret := []byte("1234567890123456789012345678901234567890123456789012345678901234")
totp = totp.New(sha256.New, secret, 8, epoch, 30)

totp.Password(time.Unix(59, 0)) // 90693936
totp.Password(time.Unix(1111111109, 0)) // 25091201
totp.Password(time.Unix(1111111111, 0)) // 99943326
totp.Password(time.Unix(1234567890, 0)) // 93441116
totp.Password(time.Unix(2000000000, 0)) // 38618901
totp.Password(time.Unix(20000000000, 0)) // 47863826
{{< /highlight >}}

## Summary

In this post, I intended to explain to you how those new-fangled passcodes that you receive in texts or emails are generated. I wanted to explain the algorithms to you and how they work so that you can implement support for them in your own applications.

{{< rawhtml >}}
<span>Photo by <a href="https://unsplash.com/@razvan_mirel?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Razvan Mirel</a> on <a href="https://unsplash.com/s/photos/lock?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
{{< /rawhtml >}}