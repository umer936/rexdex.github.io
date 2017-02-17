---
layout: project_page
title: XEX file format
permalink: /projects/recompiler/xex/
project_root: recompiler
published: false
---

## XEX format

XEX is an special container for the executables on the PowerPC based Xbox360 operating system. The main difference from the normal PE EXE files is that it contains built-in encryption and compression, so to some extent it's also an exe packer. 

Unpacking the XEX file was actually hard thing mostly because there are actually little to none informations about the format. Also the whole thing it's on the fringes of legality since most of the XEX files are encrpted and the only practical source of obtaining any XEX file is to get a stolen game from Torrents or ask somebody that has a working Xbox360 Devkit to compile you one. There are some "free floating" XEX files on the internet but they are not very usefull. Fortunatelly there were some tools released in the original Xbox materials that were published by Microsoft long time ago.


## XEX layout

| Field | Size | Offset | Description  |
|:--:|:-:|:-:|---|
| xex2 | uint32 | 0 | Magic number, 0x58455832 (XEX_) |
| module_flags | uint32 | 4 | Set of moudule flags  |
| exe_offset | uint32 | 8 | Offset to the true PE executable stored inside |
| certificate_offset | uint32 | 16 | Offset to the certificate (signed executables only) |
| header_count | uint32 | 20 | Number of XEX headers |

The module flags describe what kind of the executable we are dealing with. Usually it's 0x1 that means "TITLE", a "title" term is commonly used in Xbox nomenclature when refering to the game.

At the "exe_offset" in the file we will find the **ENCRYPTED** PE executable (more on this later).

At the "certificate_offset" we have more detailed section that contains varios additonal data (region locking, media type) but also 1024-bit RSA signature used by the retail OS to verify executable authenticity. This is probably checked by the system to validate that the game executable is "legit". The most important thing though is that the executalbe section table lies 0x180 bytes after the certificate_offset.

After the XEX header we have number of so called optional headers, the header_count indicate how much.

| Field | Size | Offset | Description  |
|:--:|:-:|:-:|---|
| key | uint32 | 0 | Magic number, sderves to identify the type of the header |
| value | uint32 | 0 | Value or offset, depends on the header type |

When the lower byte of the key *key* is 0 or 1 than the "value" is interpreted directly. If the lower byte of the *key* field is smaller than 0xFF than it indicates the size of so called "small data" that is stored directly at the offset in file given by *value*. Finally, when the lower byte of the *key* is 0xFF than we are dealing with "big data". The length of the big data is stored at the offset indicated by *value* and the rest of the big data follows.

There are following known types of the headers:

| Type | Data Size | Magic Value |
|---|---|---|
| RESOURCE_INFO | BigData | 0x000002FF |
| FILE_FORMAT_INFO | BigData | 0x000003FF |
| DELTA_PATCH_DESCRIPTOR | BigData | 0x000005FF |
| BASE_REFERENCE | SmallData | 0x00000405 |
| BOUNDING_PATH | BigData | 0x000080FF |
| DEVICE_ID | SmallData | 0x00008105 |
| ORIGINAL_BASE_ADDRESS | Value | 0x00010001 |
| ENTRY_POINT | Value | 0x00010100 |
| IMAGE_BASE_ADDRESS | Value | 0x00010201 |
| IMPORT_LIBRARIES | BigData | 0x000103FF |
| CHECKSUM_TIMESTAMP | SmallData | 0x00018002 |
| ENABLED_FOR_CALLCAP | SmallData | 0x00018102 |
| ENABLED_FOR_FASTCAP | Value | 0x00018200 |
| ORIGINAL_PE_NAME | BigData | 0x000183FF |
| STATIC_LIBRARIES | BigData | 0x000200FF |
| TLS_INFO | SmallData | 0x00020104 |
| DEFAULT_STACK_SIZE | Value | 0x00020200 |
| DEFAULT_FILESYSTEM_CACHE_SIZE| Value | 0x00020301 |
| DEFAULT_HEAP_SIZE | Value | 0x00020401 |
| PAGE_HEAP_SIZE_AND_FLAGS | SmallData | 0x00028002 |
| SYSTEM_FLAGS | Value | 0x00030000 |
| EXECUTION_INFO | SmallData | 0x00040006 |
| TITLE_WORKSPACE_SIZE | Value | 0x00040201 |
| GAME_RATINGS | Value | 0x00040310 |
| LAN_KEY | SmallData | 0x00040404 |
| XBOX360_LOGO | BigData | 0x000405FF |
| MULTIDISC_MEDIA_IDS | BigData | 0x000406FF |
| ALTERNATE_TITLE_IDS | BigData | 0x000407FF |
| ADDITIONAL_TITLE_MEMORY | Value | 0x00040801 |
| EXPORTS_BY_NAME | SmallData | 0x00E10402 |

The one that's most interesting for us is the IMAGE_BASE_ADDRESS and ENTRY_POINT. To be able to decompress the packed image we need to look at the FILE_FORMAT_INFO as well. The import table could be found in the IMPORT_LIBRARIES block.

# XEX decompression

The FILE_FORMAT_INFO block contains simple structure:

| Field | Size | Offset | Description  |
|:--:|:-:|:-:|---|
| encryption_type |	uint16 | 0 | Type of encryption (0=none, 1=normal) |
| compression_type | uint16 | 2 | Type of compression |

The compression type mostly takes 2 values: None(0), Basic (1) or Normal (2), there is also a Delta compression (3) but it's unknown how it works. The basic compression consists of blocks 


# XEX decription

Decrypting the image requires connecting few pieces of information. First, at the offset indicated by certificate_offset a following structure could be found:

| Field | Size | Offset | Description  |
|:--:|:-:|:-:|---|
| header_size | uint32 | 0 | Size of the header |
| image_size | uint32 | 4 | Size of the image data (useful) |
| rsa_signature | uint8[256] | 8 | RSA signature of the PE executable |
| image_flags | uint32 | 268 | Flags of the image itself |
| load_address | uint32 | 272 | At which memory address the PE image should be loaded |
| media_id | uint8[16] | 320 | ID of the media the executable is on |
| file_key | uint8[16] | 336 | File key that can be used to decrpt the content (but not directly) |
| game_regions | uint32 | 380 | Region lock flags  |
| media_flags | uint32 | 384 | Media flags determining source of the file |

XEX payload Rijndael cipher (now AES)

The most important field here is the *file_key*. This key than has to be decrypted with special "hidden" key 

## References

[http://www.free60.org/wiki/XEX](Free60 description of the XEX)

[https://github.com/Free60Project](Sourcecode of the Free60 project)

