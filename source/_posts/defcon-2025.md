---
title: DEFCON33 Takeaway Bento Box
date: 2025-10-10
tags: DEFCON
categories: Life
---

DEFCON33 was really fun. It was my first one, and you know what they say about firsts.

I definitely had some preconceptions about the event, that it was going to be some cyberpunky, neon-drenched event with giant flying robotic dragons everywhere and cool technical demonstrations around every corner. Only that last one was true. I'll share my thoughts on the event and my overall experience there (it was my first time in the States too), making this more of a vibes-based blog post rather than something technical.

## The United States of America

There were a lot of unknowns before this trip. I was especially worried about the general political climate in Los Angeles, but thankfully nothing out of the ordinary happened. I was also worried about racism and discrimination, but everyone I met there was fortunately far from what I imagined, with many people being really aggresively friendly. It's so different from Singapore, where it's a lot more difficult to find that same eagerness to connect and converse spontaneously.

Hard to say which environment I prefer. I do feel a lot more confident in the US - surrounded by it's people, I feel a lot more at ease when asking questions at workshops and talks, with other participants often chiming in alongside the speaker to allay my curiosity. It's a very welcoming feeling, like everyone is collaborating on improving together. With Singaporeans sometimes being too competitive, having been Pavloved by the bellcurve and the notion that sharing is sabotage, you'd be hard pressed to find something similar (I've been very fortunate to have found that environment in NUS Greyhats).

### Death Valley is called Death Valley because of DEFCON (This joke works better in audio form)

I thought we had it bad in Singapore. I was proved wrong with a slap of sauna air to the face once I left the airport. Had trouble walking anywhere when the sun was up. That's the end of this segment. I just wanted to complain about the heat.

### Okay, the actual event

Before I entered the expo hall, I had two simple goals in mind: talk with as many people as possible and fully immerse myself in every learning opportunity that presented itself. But I didn't have to think about it so consciously. Everything just came so naturally in there, I was so eager to listen and share, and conversations flowed so naturally, and everything on display was so interesting and made me want to know more.

I had planned a schedule with all the talks I wanted to visit using HackerTracker. I visited the quantum, crypto(graphy), and crypto(currency) villages in that order of frequency. Besides spending time on the red team village CTF in transit, of course.

![Aug 8 schedule](aug8sched.jpg)
![Aug 9 schedule](aug9sched.jpg)

These talks simply interested me the most, especially the quantum ones. I am most definitely not a physicist, and not even remotely knowledgeable about anything physics beyond a junior college level, only being aware of the most abstract quantum concepts like entanglement and superposition. But that's definitely why I was so intrigued. Expecting to be lost in jargon and complexity, I was instead welcomed with well-crafted, compelling presentations that were manageable to work through and left plenty of space to explore. The researchers and presenters were also excited to answer any of my questions, and even stuck around after to talk for a while. 

I have lots of respect for them, and their excitement at their profession. It was incredibly infectious, and I found myself rapt in every talk I attended. I took a lot of notes. 

![Notes](notes.png "Just one of the many pages of notes I took during the sessions")

## Speaking about the notes I took...

Here are some of the more interesting bric-a-bracs in my notebook. Keep them a secret from everyone else though. Expounding on all of these topics in this post is redundant, so I've added relevant links if you'd like to explore more.

### On the power consumption of XOR operations

[Niall Canavan](https://ieeexplore.ieee.org/document/11000140) notes that an XOR operation outputting 0 consumes less power than outputting 1, and exploits this fact to recover some secret bits from doing power analysis on parity checking. Combined with a more implicit partial information leakage by solving an SLE over GF(2), you're able to recover the full secret key somewhat reliably in the cascade error correction phase in QKD.

### On arithmetic circuits and their creative medium

While learning about ZK proof systems like Groth16, I've been creating and compiling arithmetic circuits using Circom. Anto Joseph proposes an alternative, [ZKVMs](https://rareskills.io/post/zkvm), where every instruction in some sort of intermediate representation of a language like Rust has an underlying arithmetic circuit. This would allow for the writing of circuit logic in a high level language, instead of having to worry about circuit level vulnerabilities. I was concerned about the efficiency of the resulting circuit, but after some testing it seems to be workable.

### On ZK proof systems in web2 protocols

Anto also raises [TLS Notary](https://tlsnotary.org/), which wraps TLS connections in ZK proof systems to allow for selective disclosure of HTTP requests to a verifier using MPC. Reminded me of something like a Merkle proof.

### On hardware side channel attacks

Michael Schloh von Bennewitz and Param D Pithadia had a bunch of bare hardware wallets on display, and showcased the capabilities of [ChipWhisperer](https://chipwhisperer.readthedocs.io/en/latest/) in power analysis. After asking about preventive measures, I learned about security measures such as potting, and introducing a bunch of artificial noise.

### On HSMs and reconfigurability

[Pablo Trujio](https://www.controlpaths.com/about/) introduced the idea of reconfigurable HSMs using FPGAs, with the looming threat of quantum algorithms being feasible. Keys can be stored in write only fuses, SPI can be used as a replacement for I2C for full duplex streams, and even a [TRNG](https://www.controlpaths.com/2025/07/13/generating-true-random-numbers-with-fpga/) could be implemented. I was really skeptical about this, but unfortunately this was the only talk where I didn't get to interact with the speaker, as everyone was promptly ushered out as soon as the sharing was over.

### On crosstalk in quantum chips

[Jakub Szefer](https://arxiv.org/pdf/2507.22140) imagines a future where multitenant cloud-based quantum chips are available, and proposes an attack where an attacker running circuits at the same time as another user on the same chip could run specific CNOT control pulses, maliciously exciting the user's qubits and interfering with the output probabilities. This one was very cool. It's similar to RF attacks, but now the action at a distance is even spookier. I asked about whether you could add periodic control pulses to your own legitimate circuits to mitigate the effects of any malicious circuits. That is not how quantum crosstalk works. Lol.

## Any other thoughts?

It was very nice of them to give us the black badge. The ticket prices are very expensive. Also, eating out in the US wreaks havoc on the wallet.

I came back home finding myself missing Singapore and her lushness. This place is a lot better than I give it credit for.