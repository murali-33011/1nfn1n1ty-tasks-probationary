## 3. Steganography
 
Steganography is hiding data inside other data in a way that is not visible to the naked eye. The carrier file looks completely normal. Images and audio are the most common carriers.
 
### LSB Steganography
 
Every file is made of bytes. Every byte is 8 bits. The **Least Significant Bit** is the rightmost bit in a byte. Flipping it changes the byte value by only 1, which in an image means a colour change of 1 out of 255. Completely invisible to the human eye.
 
The technique: take a secret message, convert it to binary, then replace the LSB of consecutive bytes in the image with the bits of the message one by one. To extract the message you just read the LSB of each byte in sequence.
 
### Worked Example (from my notes)
 
**The image contains this binary data (the cover bytes):**
 
```
00110010 01100011 01101111 01101111 01101100 00110100 01101101
01100101
```
 
These bytes decode to the text: `2cool4me`
 
**The secret message we want to hide is the character `y`**
 
Step 1: Convert `y` to binary:
```
y = 01111001
```
 
Step 2: Replace the LSB of each cover byte with one bit from `y`, left to right.
 
The bits of `y` are: `0  1  1  1  1  0  0  1`
 
Going through each cover byte and swapping its last bit:
 
| Cover byte | LSB of `y` | Result |
|---|---|---|
| `0011001**0**` | `0` | `00110010` (no change) |
| `0110001**1**` | `1` | `01100011` (no change) |
| `0110111**1**` | `1` | `01101111` (no change) |
| `0110111**1**` | `1` | `01101111` (no change) |
| `0110110**0**` | `1` | `01101101` (changed: 0 became 1) |
| `0011010**0**` | `0` | `00110100` (no change) |
| `0110110**1**` | `0` | `01101100` (changed: 1 became 0) |
| `0110010**1**` | `1` | `01100101` (no change) |
 
**Final stego bytes:**
```
00110010 01100011 01101111 01101111 01101101 00110100 01101100
01100101
```
 
This decodes to `2cool4me` still. Visually identical. But the LSBs now spell out `01111001` which is `y`.
 
The key note from the images: when the bit that needs to be embedded does not match the current LSB, you flip it. When it does match, you leave it alone. The image noted that one particular byte had its 0 changed to a 1 because the message bit required it.
 
### Tools
 
```bash
steghide extract -sf image.jpg
zsteg image.png
stegsolve
binwalk image.jpg
```
 
---
