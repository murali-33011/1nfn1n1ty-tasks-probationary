# picoCTF Steganography and Forensics Writeups



## Challenge 1: cat.jpg

**Category:** Metadata
**Flag:** `picoCTF{the_m3tadata_1s_modified}`

### Context

The prompt said files can be changed in a secret way. That wording is vague on purpose but on an image challenge the first place I always check is metadata. EXIF fields are a classic hiding spot because image viewers never show them and most people do not think to look there.

### What I Did

First I confirmed the file was actually a JPEG and not something renamed:

```bash
file cat.jpg
```

Came back as a real JPEG, 2560x1598. Then I ran exiftool to pull all the metadata:

```bash
exiftool cat.jpg
```

Most fields were normal camera output. Two stood out immediately. The Copyright Notice field said `PicoCTF` which no camera ever writes, and the License field had a value that started with `cGljb` which I recognised as base64 for `pico`. Decoded it:

```bash
echo "cGljb0NURnt0aGVfbTN0YWRhdGFfMXNfbW9kaWZpZWR9" | base64 -d
```

That was the flag.



## Challenge 2: garden.jpg

**Category:** File structure
**Flag:** `picoCTF{more_than_m33ts_the_3y339140129}`

### Context

The prompt said the file contains more than it seems. After metadata came back clean I needed to think about where else data could be hidden in a JPEG. A JPEG file ends at the `FF D9` End of Image marker. Image viewers stop reading at that point. But the file on disk can continue past it indefinitely and viewers will never show or report that extra content.

### What I Did

Ran the baseline checks first:

```bash
file garden.jpg
exiftool garden.jpg
binwalk garden.jpg
```

Metadata was clean, just a standard sRGB color profile. Binwalk only found the JPEG header and nothing embedded. I moved to strings to look for readable text in the raw bytes:

```bash
strings -n 6 garden.jpg | grep -i "pico\|flag"
```

That gave me the flag directly. I confirmed it precisely by finding the EOI marker in Python and printing everything after it:

```python
data = open('garden.jpg', 'rb').read()
eoi = data.rfind(b'\xff\xd9')
print(data[eoi + 2:])
```

There were 57 bytes appended after the real end of the image containing exactly the flag text. The file looked completely normal in any image viewer.



## Challenge 3: drawing_flag.svg

**Category:** Markup
**Flag:** `picoCTF{3nh4nc3d_aab729dd}`

### Context

SVG files are just XML text so there are no binary tricks here. The place to look is the markup itself. Hidden content in SVGs is often done by setting font sizes to near-zero values so text is technically present in the file but renders invisibly.

### What I Did

```bash
file drawing_flag.svg
cat drawing_flag.svg
```

The file was made in Inkscape, which leaves a lot of editor metadata. Reading through the markup I found a `<text>` element broken into a long series of `<tspan>` children, each holding one or a few characters. The styling on them was:

```
font-size:0.00352781px
```

That is a font size roughly one 280,000th of a pixel. The text was there in the file, rendered on top of the drawing, but completely invisible at any zoom level. I extracted all the tspan content and concatenated it:

```python
import re
data = open('drawing_flag.svg').read()
tspans = re.findall(r'<tspan[^>]*>([^<]*)</tspan>', data)
flag = ''.join(tspans).replace(' ', '')
print(flag)
```

That gave me the flag.



## Challenge 4: flag.txt

**Category:** File signatures
**Flag:** `picoCTF{now_you_know_about_extensions}`

### Context

A file named `.txt` that contains garbage when opened as text is a classic mislabelled file. Extensions in operating systems are just labels. The actual file format is determined by the magic bytes at the start of the file content. I needed to check what the file actually was rather than what the name claimed.

### What I Did

```bash
file flag.txt
```

Output said: `PNG image data, 1697 x 608, 8-bit/color RGB, non-interlaced`. The file was a PNG the whole time. Renamed it and opened it as an image:

```bash
mv flag.txt flag.png
```

The flag was drawn directly into the image and visible immediately on opening.



## Challenge 5: pico_flag.png

**Category:** Pixel data
**Flag:** `picoCTF{7h3r3_15_n0_5p00n_96ae0ac1}`

### Context

After all the surface checks came back clean I knew this was going to be steganography in the pixel data itself. The standard technique for hiding data in images without visible changes is LSB steganography: take the least significant bit of each color channel value across each pixel, replace it with one bit of the hidden message, and the color change is at most 1 out of 255 which is imperceptible to the eye. To extract the data you just read those LSBs back out and reassemble them into bytes.

### What I Did

First I noticed some shapes were white on a white background, only visible due to the alpha channel. I composited the image onto a dark background to check but there was nothing there beyond the normal logo design. Dead end.

Then ran the baseline checks:

```bash
file pico_flag.png
exiftool pico_flag.png
binwalk pico_flag.png
```

Plain PNG with only standard IHDR, IDAT, and IEND chunks. Nothing appended after the end marker. Went straight to LSB extraction, reading the least significant bit from the R, G, and B channels of each pixel in row order and packing those bits back into bytes:

```python
from PIL import Image
import numpy as np

arr = np.array(Image.open('pico_flag.png'))
rgb_bits = (arr[:, :, :3].flatten() & 1)
bytes_out = np.packbits(rgb_bits).tobytes()
print(bytes_out[:60])
```

Output started with `picoCTF{7h3r3_15_n0_5p00n_96ae0ac1}` followed by `$t3g0` which is just a leftover tool marker from whatever was used to embed the data.



## Challenge 6: buildings.png

**Category:** Pixel data
**Flag:** `picoCTF{h1d1ng_1n_th3_b1t5}`

### Context

Same technique as the previous challenge. After the baseline checks came back clean with nothing in metadata, nothing appended after the PNG end marker, and no extra chunks in the file, LSB extraction was the obvious next step.

### What I Did

```bash
file buildings.png
exiftool buildings.png
binwalk buildings.png
```

All clean. Ran the same LSB extraction script as challenge 5:

```python
arr = np.array(Image.open('buildings.png'))
bits = (arr[:, :, :3].flatten() & 1)
flag_bytes = np.packbits(bits).tobytes()
print(flag_bytes[:50])
```

The flag appeared at the very start of the output.



## Challenge 7: shark1.pcapng

**Category:** Network forensics
**Flag:** `picoCTF{p33kab00_1_s33_u_deadbeef}`

### Context

A packet capture with no other information. My approach with pcap files is always to start with unencrypted protocols first because they require no keys to read and often contain the relevant data in plaintext. Port 80 HTTP is the first place to look.

### What I Did

Loaded the capture with scapy since tshark was not available in the environment:

```python
from scapy.all import *
pkts = rdpcap('shark1.pcapng')
```

987 packets total. Filtered down to TCP traffic on port 80 with a payload:

```python
http_pkts = [p for p in pkts if TCP in p and (p[TCP].sport == 80 or p[TCP].dport == 80) and Raw in p]
```

Found a GET request and a 200 OK response with `Content-Length: 47`. The response body was:

```
Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}
```

Not base64, not hex. I looked at the pattern and noticed `cvpbPGS` in the position where `picoCTF` should be. Each letter was shifted by exactly 13 positions, which is ROT13:

```python
import codecs
print(codecs.decode('Gur synt vf cvpbPGS{c33xno00_1_f33_h_qrnqorrs}', 'rot_13'))
```

Output: `The flag is picoCTF{p33kab00_1_s33_u_deadbeef}`

