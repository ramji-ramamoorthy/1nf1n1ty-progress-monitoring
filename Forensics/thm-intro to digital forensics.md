# TryHackMe - Intro to Digital Forensics

**Author** Ramji R  
**Room:** [Intro to Digital Forensics](https://tryhackme.com/room/introdigitalforensics)  
**Category:** Forensics  

---

## Overview

This room introduces digital forensics concepts and walks through a practical investigation scenario. A cat named Gado has been kidnapped, and the kidnapper sends a ransom letter with an attached image. The goal is to extract metadata from both the document and the image to identify the author and the location where the photo was taken.

---

## Task 1 - Introduction to Digital Forensics

Digital forensics is the application of computer science to investigate digital evidence for a legal purpose.

Two types of investigations:

- **Public-sector** - conducted by government and law enforcement agencies as part of a crime or civil investigation
- **Private-sector** - conducted by corporate bodies investigating policy violations, either in-house or outsourced

Digital evidence can come from desktops, laptops, smartphones, cameras, USB drives, CDs, and any other digital media.

---

## Task 2 - Digital Forensics Process

### At the scene:
1. Acquire the evidence (laptops, storage devices, cameras)
2. Establish a **chain of custody** - document who handled the evidence and when
3. Place evidence in a secure container (for smartphones, block network access to prevent remote wipe)
4. Transport to the forensics lab

### At the lab:
1. Retrieve evidence from secure container
2. Create a **forensic copy** using validated software
3. Return original to secure container
4. Work only on the copy

### Key principles (from Ken Zatyko, former director of the Defense Computer Forensics Laboratory):
- Proper legal authority before starting
- Chain of custody maintained throughout
- Hash validation to confirm files are unmodified
- Use of validated tools only
- Repeatability of findings
- Final written report

**Answer:** Chain of custody

---

## Task 3 - Practical Example

### Tools used:
- `pdfinfo` - reads PDF metadata
- `exiftool` - reads EXIF data from images

### Document Metadata with pdfinfo

```bash
pdfinfo ransom-letter.pdf
```

This reveals metadata like the author, creator application, and creation date embedded in the PDF.


### Image EXIF Data with exiftool

```bash
exiftool letter-image.jpg
```

EXIF (Exchangeable Image File Format) is a standard for embedding metadata in image files. This includes:

- Camera/smartphone model
- Date and time of capture
- GPS coordinates (latitude and longitude)
- Camera settings (aperture, shutter speed, ISO)

The GPS coordinates extracted from the image:

```
GPS Position : 51 deg 30' 51.90" N, 0 deg 5' 38.73" W
```

Searching this on Google Maps or Bing Maps (replacing `deg` with `°`) reveals the street where the photo was taken: **London Bridge Street, London**.

---
