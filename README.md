# ICX-Challenge

## Challenge #1 (13th June 2019)

Announcement : 
https://twitter.com/Spl3en_ICON/status/1139219076092502018


## Solution

### Stage 1

The following link was given in the announcement :
https://bicon.tracker.solidwallet.io/transaction/0xcbeb403580fbb1e3fe43bcc5e9c6de7fa1b84ffe8e620146b53ec744a3fd6c85

The transaction 0xcbeb403580fbb1e3fe43bcc5e9c6de7fa1b84ffe8e620146b53ec744a3fd6c85 contains some data embedded in the transaction.
Such data can be copied from the browser directly, or retrieved using T-Bears.

```bash
$ tbears txbyhash 0xcbeb403580fbb1e3fe43bcc5e9c6de7fa1b84ffe8e620146b53ec744a3fd6c85 -u https://bicon.net.solidwallet.io/api/v3
```

Note that it is a Data URL, as it begins with `data:image/gif;base64` followed by base64 encoded data.
This URL can be copied directly into your browser.

The following image will be displayed : 

![challenge1.gif](challenge1.gif)

Once you have this image, saves it to your disk (Ctrl+S).

You may look at all possible pixels of that image, you won't see anything.
Indeed, the following step was hidden in the file itself, not inside the image!

There were multiple solutions to retrieve it, some people used a hexadecimal editor, some others used `strings` or even `notepad`. Ultimately, you'll see this :

![https://i.imgur.com/5ZGRlt0.png](https://i.imgur.com/5ZGRlt0.png)

Some data has been appended at the end of the file, after the TRAILER structure of the GIF format.

It is a URL to the TestNet tracker :

https://bicon.tracker.solidwallet.io/contract/cx5bd8ccf117a5eea3111085030a7310f0a94795d1

### Stage 2

The previous URL points to a SCORE (smart contract on ICON) that requires a string called `pwd` (password?) in a `get_the_money` external readonly method : https://bicon.tracker.solidwallet.io/contract/cx5bd8ccf117a5eea3111085030a7310f0a94795d1#readcontract

If you query the wrong password, it will display the following message:
![https://i.imgur.com/M5KJRI3.png](https://i.imgur.com/M5KJRI3.png)

As ICON is a blockchain and everything is transparent, we can download the SCORE source code and start figuring out what password it is expecting.

The SCORE code is the following :

https://gist.github.com/Spl3en/ed00b3bbb9f916602a3da53d7bb01592

The `get_the_money` method decrypts some encrypted data only if three "checks" methods return `True` beforehand. That decrypted data is then returned to the user.

#### Check 1

```python
    def check1(self, pwd: str) -> bool:
        return all(ord(c) < 128 for c in pwd) and len(pwd) == 11
```

The password needs to me ASCII characters only, and the length of the string is 11

#### Check 2

```python
    def check2(self, pwd: str) -> bool:
        return sum(map(ord, list(pwd))) == 947
```

The sum of the ASCII codes of the string must be equal to 947

#### Check 3

This is where it becomes interesting :


```python
    def check3(self, pwd: str) -> bool:
        v = [3707901625, 1037565863, 878818188, 1130791706, 3904355907, 2238339752, 3865851505, 252678980, 2013832146, 4251816714, 1842515611]
        for i,v in enumerate(v):
            if not self.crc32(pwd[i]):
                return False
        return True
```

That function is supposed to compute the CRC32 checksum for each character of the password, and check if it is equal to a given array stored in `v`.

But that check is actually bugged! 🤦 The method only checks if the CRC32 checksum is not equal to zero, instead of comparing against the `v` values. It was definitely not intended, I apologize for deploying the SCORE without testing if it worked well. I have wrote this challenge in less than an hour, and I definitely not wanted to spend too much time on it.

Hopefully, someone figured out this mistake and understood what needed to be done.

A simple bruteforce for each character and compare it against the `v` values would give the solution:

```

def crc32(pwd) -> bool:
    c = 0
    c = ~c & 0xffffffff
    for i in range(len(pwd)):
        c = c ^ ord(pwd[i])
    for j in range(8):
        c = (c >> 1) ^ (0xedb88320 & -(c & 1))
    c = ~c & 0xffffffff
    return c

v = [3707901625, 1037565863, 878818188, 1130791706, 3904355907, 2238339752, 3865851505, 252678980, 2013832146, 4251816714, 1842515611]
solution = ""

for c in v:
    for i in range(256):
        if crc32(chr(i)) == c:
            solution += chr(i)

print(solution)
```

That script prints the solution :
```$ python3 solve.py
ICONation<3```

![https://i.imgur.com/eeRisZ2.png](https://i.imgur.com/eeRisZ2.png)

The private key `c8a1a44c15b39e947ee4650f6b26b8d10e2d052ed39ab51fbafa04568eeebda3` can then be loaded in ICONex or Svalinn, and then use it to retrieve the funds to another address.
 
