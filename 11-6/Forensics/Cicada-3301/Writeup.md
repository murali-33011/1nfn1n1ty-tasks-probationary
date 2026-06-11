# Task 4 - TryHackMe Cicada-3301 Vol:1 (Steganography & Cryptography)


## Task 1 - Setup

Downloaded the two provided files from the TryHackMe room:

- `3301.wav` - audio file for spectrogram analysis
- `welcome.jpg` - image file for steganography



## Task 2 - Analyze the Audio

**Tool:** Sonic Visualiser

Opened `3301.wav` in Sonic Visualiser and switched to the spectrogram view. Set the frequency range to focus between 2000 Hz and 4000 Hz. A QR code became visible in the frequency domain.

**Steps:**

```
1. Open Sonic Visualiser
2. Layer > Add Spectrogram
3. Set colour scale and frequency range to 2000-4000 Hz
4. Screenshot the visible QR code
5. Scan with any QR scanner app or tool
```

Scanning the QR code returned the URL:

```
https://pastebin.com/wphPq0Aa
```

**Result:** Pastebin URL containing Base64 encoded strings.



## Task 3 - Decode the Passphrase

**Tools:** base64 (CLI), CyberChef

Opened the Pastebin link and found two Base64 encoded strings. Decoded them:

```bash
echo "SG01Ul80X1A0NTVtaHA0NTMh" | base64 -d
# Output: Hm5R_4_P455mhp453!

echo "Q2ljYWRh" | base64 -d
# Output: Cicada
```

The two outputs are:
- Plaintext: `Hm5R_4_P455mhp453!`
- Key: `Cicada`

Applied a Vigenere cipher (Beaufort variant) using `Cicada` as the key on `Hm5R_4_P455mhp453!` via CyberChef:

```
Recipe: Vigenere Decode (Beaufort variant)
Key: Cicada
Input: Hm5R_4_P455mhp453!
Output: Ju5T_4_P455phr453!
```

**Result:** Steganography passphrase = `Ju5T_4_P455phr453!`



## Task 4 - Extract from welcome.jpg

**Tool:** steghide

No useful EXIF metadata was found in `welcome.jpg`. Used steghide to attempt extraction with the recovered passphrase.

```bash
steghide extract -sf welcome.jpg
# Enter passphrase: Ju5T_4_P455phr453!
```

This extracted a file called `invitation.txt` containing:

```
https://imgur.com/a/c0ZSZga
```

Opened the Imgur link and downloaded the image `8S8OaQw.jpg`.

**Result:** `invitation.txt` extracted, `8S8OaQw.jpg` retrieved for the next stage.



## Task 5 - Extract from 8S8OaQw.jpg

**Tool:** outguess

outguess is the same steganography tool used in the original real-world Cicada 3301 challenges. Used it to extract hidden data from the downloaded image.

```bash
outguess -r 8S8OaQw.jpg bope
```

This produced a file named `bope` containing a PGP-signed message. Inside the message were two pieces of information:

1. A SHA-512 hash:
```
b6a233fb9b2d8772b636ab581169b58c98bd4b8df25e452911ef75561df649ed
c8852846e81837136840f3aa453e83d86323082d5b6002a16bc20c1560828348
```

2. Book cipher references in the format `I:1:6`, `I:2:15`, etc. (Chapter:Line:Word)

**Result:** SHA-512 hash and book cipher clues extracted from `bope`.



## Task 6 - Crack the Hash and Apply the Book Cipher

**Tools:** hashcat, rockyou.txt, book cipher logic

### Step 1 - Crack the SHA-512 hash

Used hashcat with the rockyou wordlist to crack the hash:

```bash
hashcat -m 1700 hash.txt /usr/share/wordlists/rockyou.txt
```

Alternatively, searched the hash on an online cracking database. The hash resolved to a Pastebin URL:

```
https://pastebin.com/6FNiVLh5
```

The Pastebin contained text from **"The Book of the Law"** by Aleister Crowley, Chapter I. This is the book used for the cipher.

### Step 2 - Apply the Book Cipher

The cipher indices follow the format `Chapter:Line:Word`. Positive word index counts from the start of the line; negative index counts from the end.

Chapter headers are ignored when counting lines.

Example lookups from Chapter I:
```
I:1:6  = "Had!"
I:2:15 = "pass"
```

Continuing through all the indices in the `bope` file and extracting the corresponding words spelled out a URL:

```
https://bit.ly/39pw2NH
```

**Result:** Decoded URL = `https://bit.ly/39pw2NH`



## Task 7 - The Final Song

Followed the `bit.ly` redirect:

```
https://bit.ly/39pw2NH
```

This redirected to:

```
https://soundcloud.com/user-984804993/the-instar-emergence
```

The track title is **"The Instar Emergence"**.

**Result:** The answer to the final task is `The Instar Emergence`.


## Tools Used

- Sonic Visualiser - spectrogram analysis
- steghide - JPEG steganography extraction
- outguess - alternative steganography extraction
- base64 - command line Base64 decoding
- CyberChef - Vigenere cipher decoding
- hashcat - SHA-512 hash cracking
- rockyou.txt - wordlist for hash cracking
