# TryHackMe: Intro to Digital Forensics -> Writeup

**Room:** Intro to Digital Forensics

---

## Overview

This room is a beginner-level introduction to digital forensics concepts. It covers the basic framework of how forensic investigations are conducted, what chain of custody means, and gives hands-on practice with two standard forensic tools: `pdfinfo` for document metadata and `exiftool` for image EXIF data extraction. The scenario is built around a kidnapping case where the investigator analyses a ransom letter PDF and an attached photo.

---

## Task 1: Introduction

### Question

What is the type of evidence not mentioned in the passage?

### Answer

`Laptop`

### Notes

The task introduces the types of evidence a digital forensics investigator might encounter: desktops, smartphones, digital cameras, and storage media like USB drives. A laptop is the obvious type missing from that list and is the answer by process of elimination. No tools required, just reading carefully.

This is a straightforward question but the underlying concept matters. Digital forensics is applied to a wide range of devices and the scope of an investigation is determined early on by identifying what physical and digital evidence is present at the scene.

---

## Task 2: Chain of Custody

### Question

It is essential to keep track of who is handling evidence at any point in time to ensure that evidence is admissible in court. What is the name of the documentation that would help establish that?

### Answer

`Chain of custody`

### Notes

The answer comes directly from the passage. Chain of custody is the documented record of every person who handled a piece of evidence from the moment it was collected to the moment it is presented in court. Every transfer, every access, every examination gets logged.

Why this matters in practice: if the chain of custody is broken or unaccounted for at any point, a defence lawyer can argue that the evidence may have been tampered with or contaminated. That is enough to get it thrown out regardless of how significant it is. A forensic investigator who collects evidence correctly but fails to maintain proper documentation can effectively destroy the evidentiary value of what they found.

In real investigations the chain of custody document includes the names of handlers, timestamps, locations, and the purpose of each access. It is as important as the evidence itself.

---

## Task 3: Document Metadata and Image EXIF Data

### Question 1: Who is the author of the ransom letter PDF?

**Tool used:** `pdfinfo`

```bash
pdfinfo ransom-letter.pdf
```

`pdfinfo` reads the metadata embedded in a PDF file. This includes fields like title, author, creation date, modification date, the application used to create it, and PDF version. None of this is visible when you open the PDF normally in a viewer.

**Answer:** `Ann Gree Shepherd`

The author field in the metadata revealed the name directly. This is a classic forensic mistake made by people who create documents without realising that the software embeds their identity automatically. Microsoft Word, LibreOffice, and most PDF creation tools pull the author name from the system's registered user profile and embed it in the file.

---

### Question 2: What is the name of the street where the photo was taken?

**Tool used:** `exiftool`

```bash
exiftool letter-image.jpg
```

`exiftool` extracts EXIF data from image files. EXIF is metadata that cameras and phones attach to photos automatically, including camera settings, timestamps, software used, and when location services are enabled, GPS coordinates.

The output included GPS latitude and longitude values. Taking those coordinates and looking them up using a mapping service revealed the location.

**Answer:** `Milk Street`

The GPS data embedded in the image pointed to a location that resolved to Milk Street. The kidnappers photographed something at a specific location and sent the image without stripping its metadata first. The coordinates were sitting in plain sight inside the file.

This is one of the most well-known ways people get caught through digital forensics. Smartphones in particular embed GPS coordinates into every photo by default unless the user has explicitly disabled location services. Many people have no idea this is happening.

---

### Question 3: What is the model name of the camera used to take the photo?

**Tool used:** `exiftool` with `grep`

```bash
exiftool letter-image.jpg | grep -i camera
```

The EXIF data also contains the camera make and model. Rather than reading through the full output, grepping for `camera` pulls out only the relevant line.

**Answer:** `Canon EOS R6`

The camera model is another piece of information embedded automatically. Combined with the GPS data and the author name from the PDF, the investigation now has the writer's name, the physical location where a photo was taken, and the specific camera model used to take it. Each piece adds to the picture.

---

## Key Takeaways

**pdfinfo** is a fast way to extract document metadata from PDFs. Author, creation tool, timestamps. None of this is visible in the document itself but it all lives in the file.

**exiftool** does the same for images. GPS coordinates, camera make and model, timestamps, software used to edit it. A single `exiftool` command on an image can tell you where it was taken, when, and with what device.

Both tools demonstrate the same core forensics lesson: files contain far more information than what is visible to the user. People routinely expose their identity and location without realising it because the software that creates their files is doing it for them silently.

The chain of custody concept from Task 2 ties back into why proper tool usage matters. Findings from these tools only hold up in court if the evidence was handled correctly and the analysis was documented. The technical skill and the procedural discipline both matter equally.

---
