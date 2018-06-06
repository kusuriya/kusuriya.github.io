---
layout: default
title: Curing the insomnia: enable sleep for the X1 Carbon 6th Gen
tags:
    - Hardware
    - Lenovo
    - FreeBSD
---

# Curing the insomnia: enable sleep for the X1 Carbon 6th Gen

So I recently got a great deal on an X1 Carbon 6th generation and love it, but there were a few downsides to it. The biggest down side is recently Lenovo decided to go all in on new is better and only provides S0ix sleep states. This is an issue for operating systems that do not support S0ix sleep states yet, like FreeBSD. But never fear, thanks to a few linux hackers and some creative work we can patch S3 back into our DSDT tables and be good to go.

<!-- more -->

## Up front complaints
This was fairly annoying and I actually sort of am sore at Lenovo over it. There is 0 reason to axe S1-3 states, new != better. Lenovo should do the right thing by their customers and provide a firmware update that includes the patched DSDT tables and readd S1-3 to it, even if it is an optional firmware. I am not holding my breath that Lenovo will do this after my experience with the W540 I had but IMO that is what should happen.

## The quick overview of what to do
So as a bit of a TL;DR here I will do just a quick rundown with few to no details and expand on it. I cant just hand out an already compiled patched DSDT since everyone's system is a bit different so the memory tables may be different between my X1 Carbon and yours for many reasons. So basically the TL;DR is:
1. Use acpidump to dump and decompile the ASL for your machine.
2. Find the couple instances of One in between SS3 and SS4, they cant be there and the compile will fail with em
3. Add the S3 state below S0 System State but above S4 System State
4. Compile the ASL
5. Add the custom AML to loader.conf
6. Reboot and enjoy better battery life and closing the lid and bolting

Now that seems like a bit of a whirlwind and I'll go through and explain it a bit better ahead.

## Getting your decompiled DSDT with acpidump
Thankfully this part works flawlessly and is included in FreeBSD. Really this was the easiest part of this entire thing.

```
# acpidump -d > [FileName].asl
```

That will dump and decompile your various ACPI tables into the ACPI language. Everything your computer does via ACPI will be in here but it is fairly cryptic and you can blame every computer manufacturer on the planet for that. ACPI is one of thoose things that the intention was great, the idea awesome, the execution less than stellar. If you get lucky youll get a really well documented ACPI that may contain some source code for it but dont count on it.

So the next question after you crack this open is where do I start.

## Fixing a few errors in the compilation

So now that you have the asl and all the other fun stuff ready to go there is a small edit we should make first that will allow the ASL to actually compile. In the decompilation there are a coule stray Ones floating around in the code around line 1010 for me, I will mention again the line numbers may be different for you based on configuration options and other memory tables but in every version I've seen the code block is the same.

```asl
    Name (SS1, 0x00)
    Name (SS2, 0x00)
    Name (SS3, One)
    One
    Name (SS4, One)
    One
```

Thoose ones will cause the syntax error mentioned above. They dont seem to mean anything when removed so just axe them and make the block look like

```asl
    Name (SS1, 0x00)
    Name (SS2, 0x00)
    Name (SS3, One)
    Name (SS4, One)
```

With thoose removed it is worth doing a test compile at this point and make sure that the you have a .aml file. At the time of writing this in June of 2018 there is a bug with iasl that causes some dumps to not beable to be processed by iasl, the X1C6 seems to be one of the unlucky ACPIs that gets hit. 20180313 of acpica tools includes a version of iasl that worked for me and you can snag that from https://acpica.org/downloads. I have talked with some guys with an Intel email address and they have figured out what the bug is and are working on a fix.

to do the compile:
```
/path/to/the/older/version/of/iasl [filename].asl
```

This if you get something along the lines of Segmentation Fault, or iASL:Segmentation Fault, you may have used the wrong iasl or may need to try a different version. this part can be a bit picky.

# Teaching your system to sleep

Now a bit of house keeping to do before we add the magic sauce. We need to increment the version number around line 21 so `DefinitionBlock ("", "DSDT", 2, "LENOVO", "SKL     ", 0x00000000)` becomes `DefinitionBlock ("", "DSDT", 2, "LENOVO", "SKL     ", 0x00000001)` the part at the end of the definition block is important. If you dont change 0x00000000 to any other number when you go to load in the new DSDT tables your ACPI firmware will think it already has this version and will refuse to load the modified tables. It would suck to get to the 90 yard line just to have a little detail like version number mess you up wouldn't it?

So here comes the magic. Around line 28190 there is a line `Name (\_S4, Package (0x04)  // _S4_: S4 System State` That line defines the S4 sleep state.

So before that line you want to add,
```asl
Name (\_S3, Package (0x04) // _S3_: S3 System State
{
    0x05,
    0x05,
    0x00,
    0x00
})
```

That will map the S3 sleep state to, from my understanding, S0i3 and allow your machine to suspend to ram. After you get this bit in here compile the asl file and copy it to `/boot/`.

Now with the modified DSDT tables we need to get the file in the right place and tell FreeBSD to load the new tables on boot. So first things first, copy the `[filename].aml` file from where you had your `[filename].asl` file stored to `/boot/`. It makes sense to have it here with the rest of the FreeBSD boot files since we can be sure `/boot/` will be available to the system at load. Now with the aml file in `/boot/` we just need to make a change to loader.conf

/boot/loader.conf
```
acpi_dsdt_load="yes"             # DSDT Overriding
acpi_dsdt_name="/boot/[filename].aml"
```

This change tells the kernel at load you want to load in new DSDT tables and what table to load. Once you have this all done and saved restart your laptop and activate the new tables.

## Success

Your machine should boot right up and work without any errors extra errors. Now once the machine comes all the way up we want to check and make sure that we have the new mode available. So the first stop would be sysctl

```
$ sysctl hw.acpi.supported_sleep_state
hw.acpi.supported_sleep_state: S3 S4 S5
```

If you see S3 there you are good to start the next test and as root run `zzz` or `acpiconf -s 3`, and with that your machine should suspend. Once you have suspended the system try resuming the system and if all is well and good on the resume set `hw.acpi.lid_switch_state` to S3 and add `hw.acpi.lid_switch_state=S3` to your sysctl.conf to allow the system to suspend on closing the lid.