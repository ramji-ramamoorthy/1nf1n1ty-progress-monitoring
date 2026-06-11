# TryHackMe: Cicada-3301 Vol:1

**Author** Ramji R  
**Category:** Steganography    
**Difficulty:** Medium    
**Tools:** Sonic Visualizer, CyberChef, steghide, outguess, hash-identifier, dcode.fr    

---

## Background

This room is modelled after the real Cicada 3301 challenge that ran from 2012 onwards. It chains together audio steganography, QR codes, Base64 decoding, Vigenere cipher, image steganography, and a book cipher into one multi-step puzzle.

The zip file given contains two files:

- `3301.wav` -- the audio file to analyze
- `welcome.jpg` -- the image with hidden data

Archive password: `1 kn0w 1 5h0uldn'7!`

---

## Task 1 -- Download

Downloaded and unzipped the given folder. No questions to answer here, just getting the files.

---

## Task 2 -- Analyze the Audio

Opened `3301.wav` in **Sonic Visualizer**. The waveform looks normal at first but after adding a Spectrogram layer (Layer tab --> Add Spectrogram) and tuning the settings:

- Color scheme: White on Black
- Scale: dBV2

A QR code appeared hidden in the spectrogram. Took a screenshot of it and scanned it using `zbarimg`:

```bash
zbarimg qrcode.png
```

Output:

```
url: https://pastebin.com/wphPq0Aa
```

Visiting the pastebin link revealed two Base64-encoded strings:

```
Passphrase: SG01Ul80X1A0NTVtaHA0NTMh
Key:        Q2ljYWRh
```

**Q: What is the link inside the audio?**  
`https://pastebin.com/wphPq0Aa`

---

## Task 3 -- Decode the Passphrase

### Step 1: Base64 decode both strings

```bash
echo SG01Ul80X1A0NTVtaHA0NTMh | base64 -d
# Hm5R_4_P455mhp453!

echo Q2ljYWRh | base64 -d
# Cicada
```

**Q: What is the decrypted passphrase?**  
`Hm5R_4_P455mhp453!`

**Q: What is the decrypted key?**  
`Cicada`

### Step 2: Apply the Vigenere cipher

The hint says "French Diplomat Cipher" which is the Vigenere cipher. Used https://www.dcode.fr/vigenere-cipher to encrypt the decoded passphrase `Hm5R_4_P455mhp453!` with the key `Cicada`.

Result:

**Q: What is the final passphrase?**  
`Ju5T_4_P455phr453!`

---

## Task 4 -- Gather Metadata

Used `steghide` to extract hidden data from `welcome.jpg` using the final passphrase:

```bash
steghide extract -sf welcome.jpg
# Enter passphrase: Ju5T_4_P455phr453!
# wrote extracted data to "invitation.txt"

cat invitation.txt
# https://imgur.com/a/c0ZSZga
```

The extracted file contained a link to an Imgur gallery with another image.

**Q: What link is given?**  
`https://imgur.com/a/c0ZSZga`

---

## Task 5 -- Find Hidden Files

Downloaded the image from the Imgur link (`8S8OaQw.jpg`). Tried the usual tools (steghide, binwalk, exiftool) but nothing worked. The hint says to use the same tool that was used in the original Cicada 3301 challenges, which is **outguess**.

Installed and ran it:

```bash
outguess -r 8S8OaQw.jpg output.txt
cat output.txt
```

The extracted file was a PGP-signed message containing a book cipher:

```
Welcome again.

Here is a book code. To find the book, break this hash:

b6a233fb9b2d8772b636ab581169b58c98bd4b8df25e452911ef75561df649ed
c8852846e81837136840f3aa453e83d86323082d5b6002a16bc20c1560828348

Use positive integers to go forward in the text use negative integers to go backwards.

I:1:6
I:2:15
I:3:26
...
```

**Q: What tool did you use to find the hidden file?**  
`outguess`

---

## Task 6 -- Book Cipher

### Step 1: Identify and crack the hash

Used `hash-identifier` on the hash from the PGP message:

```bash
hash-identifier b6a233fb9b2d8772b636ab581169b58c98bd4b8df25e452911ef75561df649edc8852846e81837136840f3aa453e83d86323082d5b6002a16bc20c1560828348
```

Output identified it as `SHA-512` (the `Hash: SHA1` in the PGP header was misleading -- that referred to the signature algorithm, not the hash we needed to crack).

Used an online hash cracker (https://md5hashing.net/hash) to crack it. The result was a link to another pastebin containing the book text.

**Q: What is the hash type?**  
`SHA512`

**Q: What is the link from the hash?**  
`https://pastebin.com/6FNiVLh5`

### Step 2: Apply the book cipher

The pastebin contained the text of "The Book of the Law" by Aleister Crowley. The cipher notation `I:1:6` means Chapter I, Line 1, Character 6. Negative integers go backwards from the end of the line.

Worked through each position in the cipher to extract characters and assembled the final link.

**Q: What is the link?**  
`https://bit.ly/39pw2NH`

---

## Task 7 -- The Final Song

Followed the shortened URL which redirected to a SoundCloud page containing a track.

**Q: What is the song linked?**  
`The Instar Emergence`

---
```
