---
layout: post
title: Dealing with REG_BINARY dates in Powershell
date: 2018-05-13 2:00 PM
category: blog 
tags: blog powershell
---

### Dealing with REG_BINARY dates in Powershell

So I'm in the middle of the project with a friend of mine that includes some PowerShell code to deal with various registry keys. Some of those keys include dates.
For example, the screenshot below:

![REG_BINARY date]({{ site.uri }}/assets/2018_05_13_REG_BINARY_01.png)

If you're anything like me, you have previously dealt with Microsoft's _unusual_ handling of dates - like their weird 
[FILETIME](<https://msdn.microsoft.com/en-us/library/windows/desktop/ms724284(v=vs.85).aspx>)
structure which "_Contains a 64-bit value representing the number of 100-nanosecond intervals since January 1, 1601 (UTC)_". Well, the registry key shows the name "Filetime"
which would lead one to believe this is simply that kind of structure (although the data being REG_BINARY certainly gives a person pause). Anyway - if it was truly a Filetime, then
PowerShell could make pretty quick work of it, as it has pretty good support for the FILETIME structures. For example:


![FILETIME Structure]({{ site.uri }}/assets/2018_05_13_FILETIME.png)


As we can see from the screenshot above, some attributes in Active Directory are stored in the FILETIME format including such things as 'lastlogon'.
The .Net Framework (and consequently, Powershell) has methods to convert this weird format into something a little more human-friendly AND Powershell's 
[Type Accelerators](https://blogs.technet.microsoft.com/heyscriptingguy/2013/07/08/use-powershell-to-find-powershell-type-accelerators/) makes dealing with these
easier still. 

To cut out the suspense - this data IS, more or less, a FILETIME structure, but stored in reverse order. So the last byte of the array needs to be the most significant byte when converting to a datetime.
This might sound a little confusing but the image below should clear things up:

![FILETIME Structure]({{ site.uri }}/assets/2018_05_13_DATETIME_ENCODING.png)

The first line where we define the long $orig, we see that this is the byte order in our registry. The second line, where we define $rev, shows the data in the order where we reversed the bytes. Now
if we use the [datetime] type accelerator to convert from a Filetime, we see that if we kept the original ordering - it simply fails as that is not a valid Filetime. The reverse-ordering works perfectly
correctly with accurate data.

The following code demonstrates two functions that can convert to the REG_BINARY datetime and convert from that type as well. This code will be cleaned up and improved and ultimately available in the
project my friend and I are working on (and if I remember, linked here). 

```powershell
function Convert-RegistryDateToDatetime([byte[]]$b) {

    # take our date and convert to a datetime format.
    [long]$f = ([long]$b[7] -shl 56) `
                -bor ([long]$b[6] -shl 48) `
                -bor ([long]$b[5] -shl 40) `
                -bor ([long]$b[4] -shl 32) `
                -bor ([long]$b[3] -shl 24) `
                -bor ([long]$b[2] -shl 16) `
                -bor ([long]$b[1] -shl 8) `
                -bor [long]$b[0]

    return [datetime]::FromFileTime($f)
}

function Convert-DatetimeToRegistryDate($dt) {
    [long]$ft = $dt.toFileTime()
    $arr = @()
    0..7 | ForEach-Object {
        $arr += ([byte](($ft -shr (8 * $_)) -band 0xFF))
    }
    return $arr
}

```

This has been an interesting look into how the registry stores date information. There is a lot of interesting history in the Filetime structure - why its set up the way it is, etc. However, I did not think
it necessary for this to be a history lesson. 

Enjoy the registry work!

Sam
