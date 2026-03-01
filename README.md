# LUKS2 Guide

*In this guide I aim to show how to create and use LUKS2 encrypted disk images on Linux.*

Disclaimer: This guide is provided for educational purposes only.
I am not responsible for any data loss, security breaches, or other damages that may result from following these instructions.
Always:
- Back up important data before proceeding
- Test procedures in a safe environment first
- Understand the commands you are running
- Keep your header backups secure
- Ensure you have adequate system resources for the cryptographic operations

## Introduction

To start off, I will say that I always recommend full disk encryption via LUKS2 when someone is about to install any Linux distro.
It's essential if you want the computer to be safe. For instance if you leave it somewhere and somebody plugs in a live USB or
a USB with a full system install, they can just chroot into your computer and it bypasses the user password totally.

Obviously there is a thing called secure boot, but I will not discuss that here.

How can you add an extra layer of security to important files you may ask. Since once you opened the encryption,
and e.g. you leave the laptop somewhere, people can obviously access it. 

What you can do is create secure LUKS2 containers. These will be treated as block devices, but you can move them, copy them,
delete them, etc. and you mount then whenever and however you want. In this guide I will show you how to create them and use them.

The kernel version of your Linux has to be > 4.12

## Creating the container

First you have the use the dd command. This is a very versatile command, which can be used for many things.
What I will use it for here is the creation of this block device, which by convention I will endow with the .img extension.
It doesn't actually matter what extension you give it, you don't even have to give it any.

**However, be careful with the `dd` command, because it can wipe your drives.**

To use the command, you have to run it with sudo. Here is an example:

`sudo dd if=/dev/urandom of=/path/to/LUKS2_container.img bs=1M count=1024 status=progress`

This command writes random data onto the LUKS2_container.img, with a blocksize of 1 MB (`bs=1M`) and an overall size (`count=1024`) of 1024 MB = 1 GiB,
while displaying the progress. This provides cryptographically secure randomness (required for security). You can also use

`sudo dd if=/dev/zero of=/path/to/LUKS2_container.img bs=1M count=1024 status=progress`

this will write zeros to this new container. I recommend the previous one, but this one puts less stress on the drive,
so if you are concerned about your e.g. old SSD, use this one.

The general syntax of the command is:

`dd if=<input file> of=<output file> [options]`

## Formatting to LUKS_crypto

For this step, you will need to have cryptsetup installed. If you don't have it, just run

`sudo pacman -S cryptsetup`

`sudo apt install cryptsetup`

etc. based on your distro.

Once you have it installed, you can move onto creating the LUKS2 container in it.
It is done the following way:

`sudo cryptsetup luksFormat --type luks2 --label "EncryptedBlock" --pbkdf argon2id --pbkdf-memory 4194304 --pbkdf-parallel 4 --cipher aes-xts-plain64 /path/to/LUKS2_container.img`

This will prompt you to say *YES* and then for a passphrase. I will explain each part below.
- `--type luks2`: Use LUKS2, since LUKS1 is obsolete and less secure.
- `--label "EncryptedBlock"`: Assigns a label to identify the container.
- `--pbkdf argon2id`: Uses the Argon2id algorithm, which is the current state-of-the-art, winner of the Password Hashing Competition.
- `--pbkdf-memory 4194304`: This is 4 GiB (4194304 KiB). The KDF must allocate this amount of RAM to run. This increases the cost for an attacker, as they need the same amount of memory for each guess. For that reason specialized hardware like GPUs are much less effective. Set less if you have less RAM (if you have 8 GB or less you should assign less than 4 GiB).
- `--pbkdf-parallel 4`: Uses 4 CPU threads. This makes the process more complex and harder to parallelize on an attacker's hardware.
- `--cipher aes-xts-plain64`: Use AES encryption in XTS mode.

First lets look at the cipher in more detail.
- `aes`: This stands for the **Advanced Encryption Standard**. It's a globally recognized symmetric encryption algorithm. It's efficient, secure, and many modern CPUs (both Intel and AMD) have dedicated AES instruction sets (AES-NI) that hardware-accelerate it.
- `xts`: This is the mode of operation. Think of it as how the AES cipher is applied to the entire disk. XTS (**XEX-based Tweaked CodeBook mode with CipherText Stealing**) is specifically designed for encrypting storage devices. Unlike some other modes, XTS allows the system to encrypt and decrypt any specific sector on the disk without having to read the entire disk from the start. This is crucial for performance when your OS is reading files from different locations. XTS uses a "tweak" (based on the sector number) for each disk sector. This means that even if two sectors contain identical plaintext data, they will produce completely different ciphertext. This prevents patterns from emerging on the encrypted disk, which could be exploited by an attacker.
- `plain64`: This part specifies the initialization vector (IV) handling. An IV is a random value used to ensure that encrypting the same data twice produces different results. plain64 is the standard for XTS mode in LUKS2 and simply means it uses the sector number as the tweak value (which is exactly what XTS expects).

Next up is the pbkdf, which stands for **Password-Based Key Derivation Function**.

To understand it, let's first look at the naive approach: Simple Hashing

Imagine you use a simple hash function like SHA-256 to "convert" your password into an encryption key.
- You enter your password: MyCoolPassword!
- The system computes `key = SHA-256("MyCoolPassword!")`
- This `key` is used to unlock the container

This seems logical, but it has a critical weakness: it's incredibly fast for an attacker to test. Modern hardware (especially GPUs and ASICs) can calculate billions of SHA-256 hashes per second. An attacker who gets a hold of your LUKS header can run through entire dictionaries of common passwords and every possible combination of short passwords in a trivial amount of time.

The Solution is **Key Derivation Functions (KDF)**.

A KDF, like Argon2id, is designed to be deliberately slow, memory-hard, and computationally intensive.

When you use a strong KDF (like Argon2id), an attacker who wants to test a single password guess has to spend a significant amount of their computer's resources (CPU time and RAM) to do it. What takes a legitimate user 1 second to unlock, might now take an attacker's computer several tens of seconds or even minutes per guess. This slows down a brute-force attack making it completely infeasible to crack any reasonably complex password.

Make sure you give it a really good password, I recommend at least 18 characters, uppercase, lowercase, numbers and symbols included.

After you've done this, I recommend backing up the header, it is done this way:

`sudo cryptsetup luksHeaderBackup /path/to/LUKS2_container.img --header-backup-file luks_header.bak`

Keep this on a different drive, e.g. a flash drive!!

FAQ: *Can I change the passphrase later?*

Yes, you can. This way:

`sudo cryptsetup luksChangeKey /path/to/LUKS2_container.img`

## File system inside LUKS_crypto

Once you have done all the steps above, you are ready to put a file system into the container.

First you have to unlock it:

`sudo cryptsetup luksOpen /path/to/LUKS2_container.img vault_name`

It will ask for your passphrase. This creates a mapped device at `/dev/mapper/vault_name`. 
Give it any name instead of `vault_name`.

You have to format it to e.g. `ext4` or any other file system. I will not discuss the file system types here.
Usually `ext4` fits the purpose.

`sudo mkfs.ext4 /dev/mapper/vault_name -L vault_data`

The `-L vault_data` gives the file system a label, it isn't a necessary option, but good to have for readability.

Once you formatted it, you can mount it:

`sudo mount /dev/mapper/vault_name /mnt/your_mount_point`

Replace the `/mnt/your_mount_point` with the directory where you wish to mount it (it has to be an empty folder).

**Remember that while the container is open, the data is accessible.**

Once you finished using it, unmount it:

`sudo umount /mnt/your_mount_point`

And close it:

`sudo cryptsetup luksClose vault_name`
